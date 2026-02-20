# 17_MONITORING

## Cel
Definicja metryk, alertów i dashboardów do monitorowania systemu w produkcji.

---

## 1. Stack Monitoringu

| Komponent | Rola |
|---|---|
| **Prometheus** | Zbieranie i przechowywanie metryk (scrape co 15s) |
| **Grafana** | Dashboardy i wizualizacja |
| **Alertmanager** | Reguły alertów (email, webhook, opcjonalnie SMS) |
| **Loki** (opcjonalnie v1.0) | Centralne logowanie (structured JSON) |

---

## 2. Strategia Logowania

### Format logów
Wszystkie serwisy logują w formacie **Structured JSON** (obowiązuje od **Fazy 1**, biblioteka: `structlog`):

```
{
  "timestamp": "ISO8601",
  "level": "INFO|WARNING|ERROR|CRITICAL",
  "service": "backend|satel-worker|sms-agent",
  "event": "Alarm created for panel_001",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "context": {
    "panel_id": "panel_001",
    "event_type": "ALARM",
    "user_id": "uuid"
  }
}
```

> [!IMPORTANT]
> **Każdy log MUSI zawierać `request_id` (UUID).** Generowany per request, propagowany w headerze `X-Request-ID` przez Nginx. Pozwala śledzić request przez cały pipeline. Szczegóły konfiguracji: **19_DEV_TOOLING.md, sekcja 2**.

### Poziomy logowania

| Poziom | Kiedy | Przykład |
|---|---|---|
| DEBUG | Tylko DEV | Pełna ramka binarna ETHM-1 |
| INFO | Normalny flow | "Alarm created", "User logged in" |
| WARNING | Potencjalny problem | "SMS z nieznanego numeru", "Redis reconnect" |
| ERROR | Błąd odwracalny | "CRC invalid, retry", "DB connection pool exhausted" |
| CRITICAL | Błąd nieodwracalny | "Worker lost connection > 5 min", "RabbitMQ down" |

### Retencja logów

| Środowisko | Retencja | Storage |
|---|---|---|
| DEV | 7 dni | Lokalny disk |
| STAGE | 14 dni | Lokalny disk |
| PROD | 90 dni | NAS / Loki |

---

## 3. Metryki Biznesowe

| Metryka | Typ | Opis | Alert |
|---|---|---|---|
| `alarms_total` | Counter | Łączna liczba alarmów (per source, type) | — |
| `alarms_active` | Gauge | Aktualnie otwarte alarmy (per status) | > 50 → WARNING |
| `alarm_response_time_seconds` | Histogram | Czas od NEW do IN_PROGRESS | p95 > 5min → WARNING |
| `alarm_resolution_time_seconds` | Histogram | Czas od NEW do CLOSED | p95 > 4h → WARNING |
| `bundles_created_total` | Counter | Liczba Bundli stworzonych | — |
| `events_per_minute` | Gauge | Eventy przetworzone / minutę | > 200 → INFO (burst detected) |
| `events_deduplicated_total` | Counter | Odrzucone duplikaty | — |

---

## 4. Metryki Infrastrukturalne

### Backend API (FastAPI)

| Metryka | Typ | Alert |
|---|---|---|
| `http_request_duration_seconds` | Histogram | p95 > 1s → WARNING |
| `http_requests_total` | Counter (per status code) | 5xx > 10/min → CRITICAL |
| `websocket_connections_active` | Gauge | — |
| `auth_login_failures_total` | Counter | > 20/min → CRITICAL (brute force?) |

### Satel Worker

| Metryka | Typ | Alert |
|---|---|---|
| `satel_connection_status` | Gauge (0/1 per panel) | 0 > 2min → WARNING, > 5min → CRITICAL |
| `satel_poll_duration_seconds` | Histogram | p95 > 3s → WARNING |
| `satel_reconnect_total` | Counter | > 5/h → WARNING |
| `satel_events_published_total` | Counter | — |
| `satel_crc_errors_total` | Counter | > 10/h → WARNING |

### Redis

| Metryka | Typ | Alert |
|---|---|---|
| `redis_memory_used_bytes` | Gauge | > 90% max → WARNING |
| `redis_hit_ratio` | Gauge | < 70% → WARNING |
| `redis_connected_clients` | Gauge | > 50 → WARNING |
| `redis_evicted_keys_total` | Counter | > 100/min → CRITICAL |

### RabbitMQ

| Metryka | Typ | Alert |
|---|---|---|
| `rabbitmq_queue_messages` | Gauge (per queue) | events.processing > 1000 → WARNING |
| `rabbitmq_queue_messages_dead` | Gauge | events.dead > 0 → WARNING |
| `rabbitmq_consumers` | Gauge | 0 → CRITICAL (nikt nie konsumuje!) |

### PostgreSQL

| Metryka | Typ | Alert |
|---|---|---|
| `pg_connections_active` | Gauge | > 80% pool → WARNING |
| `pg_replication_lag_seconds` | Gauge (jeśli replika) | > 30s → WARNING |
| `pg_database_size_bytes` | Gauge | > 80% disk → WARNING |

### SMS Agent

| Metryka | Typ | Alert |
|---|---|---|
| `sms_received_total` | Counter | — |
| `sms_parsed_success_total` | Counter | — |
| `sms_parsed_failure_total` | Counter | > 0 → WARNING |
| `sms_last_received_timestamp` | Gauge | > 1h ago → WARNING (modem problem?) |

---

## 5. Dashboardy Grafana

### Dashboard 1: Operations (dla OPERATOR/ADMIN)
- Aktywne alarmy per status (wykres kołowy)
- Alarmy w czasie (timeline, last 24h)
- Średni czas reakcji (gauge)
- Top 5 obiektów z alarmami

### Dashboard 2: Infrastructure (dla DevOps)
- Status połączeń Satel Worker (per panel: zielony/czerwony)
- RabbitMQ queue depth (wykres liniowy)
- Redis memory + hit ratio
- HTTP request latency (histogram)
- Error rate (5xx/min)
- **Outbox pending count** (gauge — ciche awarie)
- **Panele w stanie UNKNOWN** (counter)

### Dashboard 3: Satel Worker (dla Debug)
- Połączenia TCP per panel (status timeline)
- Poll duration per panel
- Eventy publishowane / minutę
- CRC errors
- Reconnecty
- **Panele z przypisanym Workerem vs bez** (HA-05)
- **Snapshot freshness** (HA-06)

---

## 6. SLO / SLI (Service Level Objectives)

> **Cel:** Zdefiniować mierzalne cele jakościowe systemu. Error budget = dopuszczalny margines błędów w danym okresie.

| SLO | SLI (jak mierzymy) | Cel | Error Budget (30 dni) |
|---|---|---|---|
| **Dostarczalność alarmów** | (alarmy dostarczone do operatora < 2s) / (alarmy wygenerowane) | **99.9%** | Max 43 niedostarczone alarmy |
| **Dostępność API** | (requesty HTTP non-5xx) / (requesty HTTP total) | **99.95%** | Max 0.05% = ~21 min downtime/miesiąc |
| **Zero utraconych CRITICAL** | Alarmy CRITICAL bez dostarczenia = 0 | **100%** | 0 tolerancji |
| **Czas reakcji na alarm** | P95 czasu od `first_seen` do `CLAIMING` | **< 5 min** | — |

**Implementacja:**
- Prometheus recording rules obliczają SLI co 1 min.
- Grafana dashboard "SLO Burn Rate" — wskaźnik wypalania error budget.
- Alert: jeśli burn rate > 10x → WARNING, > 30x → CRITICAL (przekroczenie budżetu w ciągu dnia).

---

## 7. Monitoring Syntetyczny (End-to-End Probe)

**Cel:** Co 5 minut sprawdzaj, czy cały tor alarmowy działa — od RabbitMQ do operatora.

```
Cron co 5 min:
1. Wstaw test event do RabbitMQ (routing_key: "satel.alarm.test", 
   payload: {type: "SYNTHETIC_TEST", test_id: UUID, timestamp: now})
2. Poczekaj max 10s
3. Sprawdź czy event pojawił się w PostgreSQL (EVENTS table z event_id = test_id)
4. Sprawdź czy WebSocket dostarczył event (metric: ws_event_delivered_test)
5. Jeśli nie → CRITICAL alert: "Synthetic probe failed — alarm pipeline broken"
6. Jeśli tak → metric: synthetic_probe_latency_seconds (histogram)
```

**Zasady:**
- Test eventy oznaczone `source: "SYNTHETIC"` — nie wyświetlane operatorowi.
- Automatyczne cleanup: test eventy kasowane po 1h.
- Alert: `synthetic_probe_success == false` przez > 10 min → CRITICAL.

---

## 8. Metryki Cichych Awarii (Silent Failure Detection)

> Scenariusze, w których system "nie działa" ale nikt nie wie. Każda metryka poniżej **MUSI** mieć zdefiniowany alert.

### MON-03: Outbox Relay Health

| Metryka | Typ | Alert |
|---|---|---|
| `outbox_pending_count` | Gauge | > 100 przez > 5 min → **CRITICAL** |
| `outbox_relay_last_poll_timestamp` | Gauge | > 30s ago → **CRITICAL** |
| `outbox_relay_publish_errors_total` | Counter | > 10/min → WARNING |

**Uzasadnienie:** Outbox Relay może paść po cichu (unhandled exception w asyncio task), a kontener Docker pozostaje "healthy". Bez tej metryki alarmy zapisane w PostgreSQL **nigdy nie trafiają do RabbitMQ**, a operator ich nie widzi.

### MON-04: Panele w Stanie UNKNOWN

| Metryka | Typ | Alert |
|---|---|---|
| `panels_in_unknown_state` | Gauge | > 0 przez > 5 min → **CRITICAL** |
| `panel_redis_ttl_remaining_seconds` | Gauge (per panel) | < 60s → WARNING |

**Uzasadnienie:** Po utracie Redis dane paneli wygasają (TTL 5 min). Dashboard pokazuje "UNKNOWN" — operator może to zignorować jako "zaraz się połączy". W rzeczywistości system jest ślepy na alarmy.

### MON-05: Anomalna Deduplikacja

| Metryka | Typ | Alert |
|---|---|---|
| `events_deduplicated_total` | Counter (per object) | > 50/h dla jednego obiektu → WARNING |
| `events_deduplicated_rate` | Rate | Wzrost > 300% vs. baseline → WARNING |

**Uzasadnienie:** Deduplikacja z granularnością 1 min może ukryć prawdziwy alarm (sensor fires → clears → fires w ciągu 60s). Anomalnie wysoki poziom może oznaczać problem z czujką LUB faktyczny incydent.

### MON-06: Opóźnienie Replikacji + UI

| Metryka | Typ | Alert |
|---|---|---|
| `pg_replication_lag_seconds` | Gauge | > 10s → WARNING, > 60s → CRITICAL |

**Akcja w UI:** Gdy `replication_lag > 10s`:
- Raporty i eksporty generowane z repliki wyświetlają baner: "⚠️ Dane mogą być opóźnione o {lag}s"
- Header API: `X-Data-Freshness: stale` (pozwala klientowi wyświetlić ostrzeżenie)

### MON-07: Przepełnienie Bufora WebSocket

| Metryka | Typ | Alert |
|---|---|---|
| `ws_replay_overflow_count` | Counter | > 0 → WARNING |
| `ws_events_buffer_size` | Gauge | > 4500 (z max 5000) → WARNING |

> **CS-01 fix:** Bufor Redis Sorted Set `ws:events:buffer` ma max **5000** wpisów (ADR-014, Tier 1). Próg alertu skorygowany z błędnego `> 900 (max 1000)` na `> 4500 (max 5000)`.

**Uzasadnienie:** Operator traci najstarsze eventy z bufora WebSocket bez świadomości. System zwraca `replay_overflow`, ale bez metryki nikt nie wie jak często to się zdarza.

---

## 9. Reguły Alertowania

### Severity Levels

| Severity | Akcja | Kanał |
|---|---|---|
| **CRITICAL** | Natychmiast! Budzi w nocy. | SMS + Email + Dashboard |
| **WARNING** | Reakcja w godzinach pracy. | Email + Dashboard |
| **INFO** | Do wiadomości. | Tylko Dashboard |

### Reguły

| Reguła | Warunek | Severity | Opis |
|---|---|---|---|
| SatelWorkerDown | `satel_connection_status == 0` > 5min | CRITICAL | Worker stracił łączność z centralą |
| RabbitMQConsumerMissing | `rabbitmq_consumers == 0` > 2min | CRITICAL | Backend nie konsumuje eventów |
| HighErrorRate | `http_5xx_rate > 10/min` > 5min | CRITICAL | Masowe błędy API |
| **OutboxStuck** | `outbox_pending_count > 100` > 5min | **CRITICAL** | Outbox Relay nie publikuje — alarmy nie docierają |
| **PanelsUnknown** | `panels_in_unknown_state > 0` > 5min | **CRITICAL** | Brak danych o stanie paneli — system ślepy |
| **SyntheticProbeFailed** | `synthetic_probe_success == false` > 10min | **CRITICAL** | Cały tor alarmowy nie działa |
| **PanelUnassigned** | `satel_panels_without_worker > 0` > 60s | **CRITICAL** | Panel bez Workera — brak monitoringu |
| AlarmNotHandled | `alarm_response_time > 5min` | WARNING | Alarm nowy bez reakcji |
| RedisMemoryHigh | `redis_memory > 90%` | WARNING | Redis bliski limitu |
| QueueBacklog | `rabbitmq_queue_messages > 1000` > 5min | WARNING | Kolejka rośnie — backend nie nadąża |
| SMSModemSilent | `sms_last_received > 1h` | WARNING | Modem nie odbiera |
| DiskSpaceLow | `disk_used > 85%` | WARNING | Mało miejsca na dysku |
| **DedupAnomaly** | `events_deduplicated_rate > 300% baseline` | WARNING | Anormalna deduplikacja — sprawdź źródło |
| **ReplicationLag** | `pg_replication_lag > 10s` > 5min | WARNING | Replika opóźniona — raporty nieaktualne |
| **WSBufferFull** | `ws_events_buffer_size > 4500` | WARNING | Bufor blisko przepełnienia (max 5000) |

### Reguły Wyciszania Alertów (Alert Suppression)

> **Problem:** Jeśli PostgreSQL padnie, Backend generuje błędy, Outbox Relay nie publikuje, kolejka RabbitMQ rośnie — 5+ alertów jednocześnie o tym samym problemie.

| Reguła nadrzędna | Wycisza | Czas wyciszenia |
|---|---|---|
| `PostgresDown` | HighErrorRate, OutboxStuck, QueueBacklog, ReplicationLag | 15 min |
| `RedisDown` | PanelsUnknown, RedisMemoryHigh | 10 min |
| `RabbitMQDown` | RabbitMQConsumerMissing, QueueBacklog | 10 min |

---

## 10. Dyżury i Eskalacja (On-Call)

### Rotacja Dyżurów

| Element | Konfiguracja |
|---|---|
| Narzędzie | PagerDuty lub Opsgenie (do decyzji) |
| Rotacja | Tygodniowa, 2 osoby (primary + backup) |
| Godziny | 24/7 dla CRITICAL, godziny pracy dla WARNING |
| Ack timeout | CRITICAL: 10 min (brak ack → eskalacja na backup) |
| Kontakt | PRIMARY: telefon + SMS, BACKUP: email + push |

### Eskalacja

| Poziom | Czas | Kto | Kanał |
|---|---|---|---|
| L1 | 0-10 min | On-call primary | SMS/Telefon (auto z PagerDuty) |
| L2 | 10-30 min | On-call backup | SMS/Telefon |
| L3 | > 30 min | MASTER (właściciel) | Telefon bezpośredni |

### Post-Incident

Po każdym incydencie CRITICAL stosujemy blameless postmortem (szablon: `18_RUNBOOK.md`, sekcja RUN-05).
