# 09_HA_RTO_RPO

## Cel
Definicja wymagań dostępności (High Availability) oraz celów odzyskiwania (Disaster Recovery).

---

## 1. Definicje
- **RTO (Recovery Time Objective):** Maksymalny akceptowalny czas postoju systemu.
- **RPO (Recovery Point Objective):** Maksymalna ilość danych, jaką możemy stracić (w czasie).

## 2. Tabela Celów

| Komponent | RTO (Cel) | RPO (Utrata Danych) | Strategia | Automatyzacja |
|---|---|---|---|---|
| **PostgreSQL (Primary)** | **< 30s** | < 5 min (async replication) | Patroni + etcd + PgBouncer | ✅ Automatyczny failover |
| **Backend API** | **< 15s** | 0 (stateless) | 2 instancje za reverse proxy | ✅ Health-check routing |
| **Satel Worker** | **< 60s** | 0 (eventy buforowane w RabbitMQ) | Auto-restart + leader election | ✅ Redis lock |
| **RabbitMQ** | < 1 min | 0 (durable queues na dysku) | Restart kontenera (MVP) / Quorum queues (v2.0) | ⚠️ Auto-restart |
| **Redis** | < 30s | 0 (cache, odbudowywany) | Auto-restart + fallback do PG | ✅ Auto-restart |
| **Modem SMS** | < 2h | 0 (SMS zostaje u operatora) | Zapasowy modem fizyczny | ❌ Manualny |
| **Pliki (Storage)** | < 4h | < 24h | Rsync na NAS raz dziennie | ⚠️ Cron |

---

## 3. Strategia HA (High Availability)

### HA-01: PostgreSQL — Automatyczny Failover (Patroni)

**Problem:** Ręczne przełączenie na replikę wymaga obecności admina. Przy awarii o 3 w nocy RTO < 1h jest nierealne.

**Rozwiązanie:** Patroni — zautomatyzowany failover manager dla PostgreSQL.

```
┌─────────────────────────────────────────────────────────┐
│                    Klaster Patroni                       │
│                                                         │
│  ┌──────────┐     ┌──────────┐     ┌──────────────┐    │
│  │ PG Node 1│────▶│ PG Node 2│     │ etcd         │    │
│  │ (primary) │◀───│ (replica) │     │ (3 instancje │    │
│  └──────────┘     └──────────┘     │  lub 1 MVP)  │    │
│       ▲                             └──────────────┘    │
│       │                                                 │
│  ┌────┴──────────┐                                      │
│  │ PgBouncer     │ ← VIP (pojedynczy DATABASE_URL)      │
│  │ (conn pooler) │                                      │
│  └───────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

| Element | Konfiguracja |
|---|---|
| Patroni | 2 nody PostgreSQL (primary + standby) |
| etcd | Distributed consensus store (1 node MVP, 3 nody PROD) |
| PgBouncer | Connection pooler z auto-wykrywaniem primary |
| Failover | Automatyczny: primary pada → etcd wykrywa → Patroni promuje replikę → PgBouncer przełącza ruch. **Czas: < 30 sekund.** |
| Split-brain protection | etcd quorum. Jeśli primary straci kontakt z etcd → dobrowolna degradacja do read-only (STONITH). |
| Backend | Jeden `DATABASE_URL` wskazujący na PgBouncer. Backend nie musi wiedzieć, który node jest primary. |

**Ochrona przed split-brain:**
- Patroni używa `watchdog` API — jeśli node straci lock w etcd, PostgreSQL jest **automatycznie zatrzymywany** (nie ryzykuje dual-write).
- PgBouncer kieruje cały ruch RW do primary (wykrytego przez etcd), ruch RO do repliki.

**Referencja deployment:** `11_DEPLOYMENT.md`, sekcja Docker Compose.

---

### HA-02: Backend API — Multi-Instance + Reverse Proxy

**Problem:** Pojedyncza instancja Backend = Single Point of Failure.

**Rozwiązanie:** Minimum 2 instancje Backend za Nginx / Traefik.

```
                    ┌──────────────┐
                    │   Nginx /    │
Klient ──TLS──▶    │   Traefik    │
                    │ (VPN / LAN)  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼                         ▼
       ┌─────────────┐          ┌─────────────┐
       │  Backend #1  │          │  Backend #2  │
       │  /healthz ✅  │          │  /healthz ✅  │
       └─────────────┘          └─────────────┘
```

| Element | Konfiguracja |
|---|---|
| Instancje | Min. 2 (backend-1, backend-2) |
| Reverse proxy | Nginx lub Traefik z health-check (`/readyz` co 5s) |
| Routing REST | Round-robin (stateless — każda instancja obsłuży każdy request) |
| Routing WebSocket | **Sticky sessions** (`ip_hash` w Nginx lub sticky cookies w Traefik) |
| Failover | Jeśli instancja pada → reverse proxy usuwa ją z puli w < 10s |
| Deploy (rolling) | Restart instancji po kolei: `docker restart backend-1`, poczekaj na `/readyz`, potem `docker restart backend-2` |

---

### HA-03: Health Check Endpoints

Każdy serwis **MUSI** eksponować endpointy zdrowia:

| Endpoint | Cel | Co sprawdza | Zastosowanie |
|---|---|---|---|
| `GET /healthz` | **Liveness** | Czy proces żyje, czy event loop odpowiada | Docker HEALTHCHECK, Kubernetes liveness probe |
| `GET /readyz` | **Readiness** | Czy PostgreSQL, Redis, RabbitMQ są połączone | Reverse proxy routing, deployment gates |

**Specyfikacja odpowiedzi:**

```json
// GET /healthz → 200 OK
{ "status": "healthy", "uptime_seconds": 3600 }

// GET /readyz → 200 OK (lub 503 jeśli nie gotowy)
{
  "status": "ready",
  "checks": {
    "postgresql": { "status": "up", "latency_ms": 2 },
    "redis": { "status": "up", "latency_ms": 1 },
    "rabbitmq": { "status": "up", "latency_ms": 3 }
  }
}

// GET /readyz → 503 Service Unavailable
{
  "status": "not_ready",
  "checks": {
    "postgresql": { "status": "up", "latency_ms": 2 },
    "redis": { "status": "down", "error": "Connection refused" },
    "rabbitmq": { "status": "up", "latency_ms": 3 }
  }
}
```

**Referencja API:** `10_API_HIGH_LEVEL.md`.

---

### HA-04: Graceful Shutdown

**Problem:** `docker compose up --force-recreate` zabija kontenery natychmiast — aktywne połączenia WebSocket i in-flight requesty HTTP są tracone.

**Rozwiązanie:** `SIGTERM` handler w FastAPI.

```python
# Graceful shutdown flow
async def shutdown_handler():
    # 1. Przestań akceptować nowe połączenia
    server.close()
    
    # 2. Zamknij WebSocket sesje z kodem 1012 (service_restart)
    for ws in active_connections:
        await ws.close(code=1012, reason="Server restarting")
    
    # 3. Poczekaj na zakończenie in-flight requestów (max 10s)
    await asyncio.wait_for(drain_requests(), timeout=10)
    
    # 4. Zamknij połączenia z bazami danych
    await db_pool.close()
    await redis.close()
```

**Docker Compose:**
```yaml
services:
  backend:
    stop_grace_period: 15s  # daj 15s na graceful shutdown
    # Docker wyśle SIGTERM, po 15s SIGKILL
```

**Klient (Flutter):** Po otrzymaniu WS close code `1012` → automatyczny reconnect z ticket-based auth (patrz `08_SECURITY_AND_ROLES.md`, sekcja 4.3a).

---

### HA-05: Satel Worker — Leader Election i Przypisanie Paneli

**Problem:** Jeśli Worker pada, nie ma mechanizmu przypisania jego paneli do innej instancji.

**Rozwiązanie:** Redis-based leader election per panel.

```
┌─────────────────────────────────────────────┐
│           Panel Assignment (Redis)          │
│                                             │
│  panel:lock:PAT001 → worker-1 (TTL 30s)    │
│  panel:lock:PAT002 → worker-1 (TTL 30s)    │
│  panel:lock:PAT003 → worker-2 (TTL 30s)    │
│                                             │
│  Worker odświeża lock co 10s (heartbeat)    │
│  Jeśli lock wygaśnie → inny Worker przejmuje│
└─────────────────────────────────────────────┘
```

| Element | Konfiguracja |
|---|---|
| Lock | `SET panel:lock:{panel_id} {worker_id} EX 30 NX` |
| Heartbeat | Co 10s: `EXPIRE panel:lock:{panel_id} 30` |
| Failover | Worker-2 co 5s skanuje niezablokowane panele → przejmuje |
| Konfiguracja | Tabela `panel_assignments` w PostgreSQL: `(panel_id, preferred_worker_id)` |
| Metryka | `satel_worker_assigned_panels` — alert jeśli panel bez Workera > 60s |

**Kolejność startu:**
1. Worker startuje → czyta `panel_assignments` z PostgreSQL.
2. Próbuje zablokować swoje przypisane panele (Redis NX).
3. Nawiązuje połączenie TCP z ETHM-1 dla każdego zablokowanego panelu.
4. Co 10s — odświeża locki.

---

### HA-06: Persystencja Stanu Paneli (Snapshot)

**Problem:** Po utracie Redis (restart, TTL) Worker nie ma bazy do generowania eventów delta po reconnect.

**Rozwiązanie:** Tabela `panel_state_snapshots` w PostgreSQL (szczegóły: `04_DATA_MODEL_ERD.md`).

**Przepływ:**
1. Co 60s: Worker zapisuje pełny stan centrum do `panel_state_snapshots` (z kolumną `snapshot_sequence_id` BIGINT, monotonic).
2. Po reconnect do ETHM-1: Worker pobiera aktualny stan.
3. Porównuje z:
   - **Priorytet 1:** Redis (jeśli dostępny i dane nie wygasły).
   - **Priorytet 2:** Ostatni snapshot z PostgreSQL (jeśli Redis pusty).
   - **Priorytet 3:** Pusty baseline (= traktuj cały stan jako nowy — generuj pełen dump eventów).
4. **Delta Dampening Window (przy użyciu Priorytetu 2 — stale snapshot):**

> [!WARNING]
> Snapshot z PostgreSQL może być stary do 60s. Porównanie z aktualnym stanem centrali może wygenerować fałszywe eventy delta (np. strefa przeszła alarm→ok→alarm w ciągu 60s — snapshot "ok", aktualny "alarm" = fałszywy "nowy alarm").

   - Worker wchodzi w **okno tłumienia** (configurable: `SNAPSHOT_DAMPENING_SECONDS = 10`):
     a. Worker pobiera aktualny stan z ETHM-1 (Full State Dump) — **Poll #1**.
     b. Worker czeka jeden pełny cykl pollingu (1-2s) i pobiera stan ponownie — **Poll #2**.
     c. **Tylko diff między Poll #1 a Poll #2** generuje eventy (= faktyczne tranzycje).
     d. Diff między stale snapshot a Poll #1 jest **logowany** ale **NIE publikowany** jako eventy (traktowany jako "state catch-up").
5. Generuje eventy delta i publikuje do RabbitMQ.

---

### HA-07: RabbitMQ — Quorum Queues (v2.0)

**MVP:** Pojedynczy node RabbitMQ z durable queues. Akceptowalne ryzyko — restart kontenera < 1 min.

**v2.0:** Klaster 3 nodeów z Quorum Queues:
- Replikacja wiadomości na 2/3 nodów.
- Przeżywa awarię jednego noda bez utraty wiadomości.
- Jednocześnie eliminuje potrzebę disk-based persistence (dane w RAM repliki).

---

### HA-08: Redis Sentinel (v2.0)

**MVP:** Pojedynczy Redis z `restart: always`. Fallback do PostgreSQL (wolniejsze, ale działa).

> [!IMPORTANT]
> **WebSocket Replay przy awarii Redis:** Dzięki Tiered Catch-Up (patrz **10_API_HIGH_LEVEL.md, sekcja 8.5**), restart Redis NIE powoduje utraty replay. Backend automatycznie przełącza się na Tier 2 (PostgreSQL, tabela `outbox`), co zapewnia catch-up do 2h wstecz nawet bez Redisa.

**v2.0:** Redis Sentinel z 3 instancjami:
- Automatyczny failover na replikę.
- Sentinel monitoruje Master i promuje Slave jeśli Master padnie.
- Konfiguracja: `sentinel monitor stam-redis master-host 6379 2` (quorum: 2/3).

---

### HA-09: Izolacja Sieci ETHM-1 (VLAN)

**Problem:** Ruch Worker ↔ ETHM-1 (TCP port 10004) nie jest izolowany. Atakujący na LAN może:
- Przejąć jedyne połączenie TCP z ETHM-1 (limit 1).
- Wstrzyknąć ramki binarne (brak uwierzytelniania w protokole ETHM-1).

**Wymagana konfiguracja:**

| Element | Spec |
|---|---|
| VLAN dla ETHM-1 | Dedykowany VLAN (np. VLAN 100) na switchu zarządzalnym |
| Reguły firewall | Tylko IP Workera (serwer Docker) może łączyć się z portami 10004 paneli |
| ARP poisoning protection | Dynamic ARP Inspection (DAI) na switchu |
| Monitoring | Alert jeśli nowy MAC pojawi się na VLAN 100 |

**Docker Compose (macvlan):**
```yaml
networks:
  satel_vlan:
    driver: macvlan
    driver_opts:
      parent: eth1.100  # interfejs VLAN 100
    ipam:
      config:
        - subnet: 192.168.100.0/24
          gateway: 192.168.100.1

services:
  satel-worker:
    networks:
      - satel_vlan
      - internal  # wewnętrzna sieć Docker
```

---

### Ciągłość Pracy Klienta (Offline + Degraded Mode)

**Offline Read-Only (brak połączenia z Backend):**
- Można przeglądać listę obiektów (z cache SQLite).
- Można sprawdzić adres i telefon (z cache).
- Nie można odbierać nowych alarmów.
- **Przyciski mutacji (Claim, Ack, Resolve, Close) są WYŁĄCZONE.**
- Operator może zapisywać lekkie **intencje** (Intent Queue) — arbitrowane przez backend po reconnect. Szczegóły: **03_FUNCTIONAL_MODULES.md, sekcja 19.4**.
- Wyświetla się wielki czerwony baner "BRAK POŁĄCZENIA Z SERWEREM".

**Degraded Mode (Backend działa, ale Redis/RabbitMQ nie):**
- Alarmy nadal trafiają do operatora (przez PostgreSQL, wolniej).
- Baner pomarańczowy: "TRYB AWARYJNY — dane mogą być opóźnione".
- Stan live central niedostępny (Redis down) — sekcja "Stan centrali" wyszarzona.

**WebSocket Reconnect:**
- Strategia: Exponential backoff 3s → 6s → 12s → 30s (cap). **Brak limitu prób** (rzucenie po 10 próbach = klient odcięty na czas naprawy).
- Po reconnect: Ticket-based auth → replay_request z `last_sequence_id` → catch-up eventów.

---

## 4. Disaster Recovery (DR)

### Procedura (spalenie serwerowni / całkowita utrata)

| Krok | Akcja | Szacowany czas |
|---|---|---|
| 1 | Nowy sprzęt / VM | 1-4h (zależy od dostępności) |
| 2 | Instalacja Docker + Docker Compose | 15 min |
| 3 | Pobranie repozytorium kodu | 5 min |
| 4 | Pobranie backupu PostgreSQL z NAS/Cloud (Wal-G) | 15-60 min |
| 5 | Restore bazy danych | 10-30 min |
| 6 | Wgranie Docker Secrets (z sejfu/offline backup) | 5 min |
| 7 | `docker compose up -d` | 2 min |
| 8 | Satel Worker: auto-reconnect do central | < 1 min |
| 9 | Redis: cache warmup z PostgreSQL + fresh poll | < 1 min |
| 10 | Smoke test: login, lista alarmów, stan central | 5 min |

**Szacowany RTO DR: 2-6 godzin** (głównie ograniczony dostępnością sprzętu).

### Harmonogram Testów DR

| Częstotliwość | Test | Opis |
|---|---|---|
| Co miesiąc | Failover PostgreSQL (Patroni) | Zabij primary → weryfikuj auto-promocję → sprawdź Backend |
| Co kwartał | Full DR drill | Restore z backupu na pustym serwerze → smoke test |
| Co pół roku | Chaos test | Losowe zabijanie kontenerów podczas testu obciążeniowego |

### Wymóg: UTC + NTP

> **Wszystkie serwery i kontenery MUSZĄ** korzystać z UTC jako strefy czasowej i synchronizować czas przez NTP. Dryft zegarowy między serwerami (np. Backend vs. Auto-Arm cron) może powodować błędne okna cooldownu i niespójne timestampy w AUDIT_LOG.

**Konfiguracja Docker Compose:**
```yaml
services:
  backend:
    environment:
      - TZ=UTC
  satel-worker:
    environment:
      - TZ=UTC
```

**Host:** `timedatectl set-timezone UTC && systemctl enable systemd-timesyncd`
