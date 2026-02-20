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

| Serwis | Obraz | Rola | Zależności |
|---|---|---|---|
| **nginx** | nginx:alpine | Reverse proxy, TLS, health-check routing | backend-1, backend-2 |
| **backend-1** | stam-replacer-backend | API FastAPI, logika biznesowa, WebSocket (inst. 1) | pgbouncer, rabbitmq, redis |
| **backend-2** | stam-replacer-backend | API FastAPI, logika biznesowa, WebSocket (inst. 2) | pgbouncer, rabbitmq, redis |
| **outbox-relay-1** | stam-replacer-backend | Relay Outbox Pattern → RabbitMQ (instancja 1) | pgbouncer, rabbitmq |
| **outbox-relay-2** | stam-replacer-backend | Relay Outbox Pattern → RabbitMQ (instancja 2) | pgbouncer, rabbitmq |

> [!NOTE]
> **Multi-instance Outbox Relay:** Dwie instancje relay eliminują SPOF. Bezpieczne dzięki `FOR UPDATE SKIP LOCKED` — obie instancje konkurują o wiersze bez ryzyka podwójnego przetworzenia. Metryka: `outbox_relay_active_instances` (alert jeśli <1 instancja aktywna >30s).
| **satel-worker** | stam-satel-worker | Połączenie TCP/IP z centralami ETHM-1 | pgbouncer, rabbitmq, redis |
| **pg-primary** | postgres:15 (Patroni) | Primary PostgreSQL (read-write) | etcd |
| **pg-replica** | postgres:15 (Patroni) | Standby PostgreSQL (read-only) | etcd, pg-primary |
| **etcd** | bitnami/etcd | Consensus store dla Patroni failover | — |
| **pgbouncer** | pgbouncer | Connection pooler z auto-wykrywaniem primary | pg-primary, pg-replica |
| **redis** | redis:7-alpine | Live State Cache (stan central, sesje) | — |
| **rabbitmq** | rabbitmq:3-management | Kolejka zdarzeń i komend | — |
| **sms-agent** | stam-sms-agent | Odbiór SMS z modemu GSM | rabbitmq |

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
