# 16_TESTING_STRATEGY

## Cel
Opis co, jak i kiedy testujemy. Ten dokument definiuje quality gates dla CI/CD pipeline.

---

## 1. Piramida Testów

| Warstwa | Cel | Coverage Target | Kto pisze |
|---|---|---|---|
| **Unit Tests** | Logika biznesowa, parsery, walidacje | 80% (Backend), 70% (Frontend) | Developer |
| **Integration Tests** | API ↔ DB, Worker ↔ Redis, Worker ↔ RabbitMQ | Krytyczne ścieżki | Developer |
| **E2E Tests** | Full flow: login → alarm → close | 5 smoke scenarios | Developer / QA |
| **Performance Tests** | Throughput, latency pod obciążeniem | Benchmarks per endpoint | DevOps |
| **Security Tests** | OWASP Top 10, auth bypass, injection | Checklist per release | Security / DevOps |

---

## 2. Unit Tests

### Backend (Python / Pytest)
- **Alarm Lifecycle:** Testuj każde przejście stanów (NEW→IN_PROGRESS, ACK→RESOLVED, etc.)
- **Bundling Logic:** Testuj grupowanie eventów, re-trigger, timeout
- **SMS Parser (Efento):** 5 fixture SMSów (alarm, powrót, edge cases: brak temperatury, nieznany format)
- **SMS Parser (Bluelog):** 5 fixture SMSów (alert, koniec alertu, edge cases)
- **Deduplikacja:** Testuj generację dedup_key, duplikat vs nowy event
- **Auth:** Testuj JWT generation, validation, expiration, role extraction
- **Permission Matrix:** Testuj każdą kombinację rola × uprawnienie
- **Error Handling:** Testuj format odpowiedzi błędów

### Frontend (Flutter / Widget Tests)
- **Alarm List Widget:** Renderowanie stanów, kolorów, animacji
- **Login Form:** Walidacja, error messages
- **Timed Reveal:** Countdown, auto-hide po 60s
- **WebSocket Reconnect:** Obsługa disconnect/reconnect

---

## 3. Integration Tests

### Backend ↔ PostgreSQL
- CRUD obiektów z geokodowaniem
- INSERT event + bundling w jednej transakcji
- AUDIT_LOG zapis po każdej akcji

### Backend ↔ Redis
- Odczyt live state z Redis
- Fallback na direct DB query (Redis down)
- Token blacklist (logout)

### Backend ↔ RabbitMQ
- Konsumpcja wiadomości z events.processing
- DLQ — wiadomość po 3 retry trafia do events.dead
- Publikacja komendy do commands.satel

### Satel Worker ↔ Symulator
- TCP connect / handshake
- Polling komend 0x00, 0x09, 0x17
- Reconnect po disconnect
- Zapis stanu do Redis

### Środowisko testowe
- Docker Compose profile `test` z izolowanymi kontenerami
- Automatyczny setup/teardown per test session
- Fixtures: `conftest.py` z seed data

---

## 4. E2E Tests — Krytyczne Ścieżki

### Smoke Test Suite (5 scenariuszy, run na każdym deploy)

| # | Scenariusz | Opis |
|---|---|---|
| E2E-001 | Login → Dashboard | Logowanie → widzę pustą listę alarmów |
| E2E-002 | Full Alarm Flow | Event → Bundle → Claim → ACK → Resolve → Close |
| E2E-003 | Temp Alarm + Note | SMS temp event → Bundle (requires_note) → zamknij z notatką |
| E2E-004 | Object CRUD | Dodaj obiekt → edytuj → sprawdź na liście |
| E2E-005 | Secret Reveal | Pokaż hasło → countdown → auto-hide → sprawdź AUDIT_LOG |

---

## 5. Satel Worker Testing

### Symulator Centrali
- Mock TCP Server (Python `asyncio.start_server`)
- Symuluje odpowiedzi na komendy 0x00-0x20
- Scenariusze: normalny polling, zerwanie połączenia, timeout, CRC error

### Test Fixtures
- Plik z predefiniowanymi ramkami binarnymi (request/response pairs)
- Edge cases: malformed frame, partial frame, double 0xFE (byte-stuffing)

---

## 6. SMS Parser Testing

### Fixture SMSów

| # | Źródło | Typ | Treść | Oczekiwany wynik |
|---|---|---|---|---|
| 1 | Efento | Alarm | `2026-02-10 11:03:00 Alarm! Regula Leg_szczep_prawa_MIN...` | TEMP_ALARM, CRITICAL |
| 2 | Efento | Powrót | `2026-02-10 11:24:00 Powrot do normalnego stanu...` | TEMP_NORMAL, INFO |
| 3 | Efento | Edge | Brak wartości temperatury | Parsuj z null temperature_value |
| 4 | Bluelog | Alert | `Alert temp. 21040DD5 (S1, Gad Spedycja): 15.3C < 15.5C` | TEMP_ALARM, CRITICAL |
| 5 | Bluelog | Koniec | `Koniec alertu temp. dla rejestratora...` | TEMP_NORMAL, INFO |
| 6 | Nieznany | — | `Twoj kod to 123456` | Ignoruj, log WARNING |

---

## 7. Performance Testing

### Benchmarks (cele MVP)

| Metryka | Target | Narzędzie |
|---|---|---|
| GET /api/alarms (lista) | < 200ms (p95) | locust / k6 |
| POST /api/alarms/{id}/claim | < 100ms (p95) | locust / k6 |
| Alarm processing (event → bundle) | < 500ms end-to-end | Custom timer per event |
| WebSocket delivery (event → ekran) | < 1s | Custom timer per event |
| Burst 200 events w 1 min | Bez utraty, QueueDepth < 1000 | RabbitMQ monitoring |
| Concurrent operators | 10 userów bez degradacji | locust |

---

## 8. Security Testing Checklist

| # | Test | Kategoria |
|---|---|---|
| 1 | SQL Injection na /api/objects?query= | OWASP A03 |
| 2 | Broken Auth — expired token nadal działa? | OWASP A07 |
| 3 | IDOR — /api/alarms/{id} z cudzym ID | OWASP A01 |
| 4 | Rate limit — >5 login failures / min | Brute-force |
| 5 | XSS — notatka z `<script>` | OWASP A07 |
| 6 | Secret Reveal — czy hasło znika z response po 60s? | Custom |
| 7 | WebSocket — połączenie bez ważnego tokena | Auth bypass |
| 8 | File Upload — upload .exe / oversized file | OWASP A08 |

---

## 9. CI/CD Quality Gates

### PR Merge Requirements
1. ✅ Wszystkie unit testy zielone
2. ✅ Coverage nie spadł poniżej threshold
3. ✅ Linter clean (ruff + flutter analyze)
4. ✅ No security warnings (bandit / safety)
5. ✅ Code review approved

### Deploy to STAGE
1. ✅ PR merged do main
2. ✅ Integration tests zielone
3. ✅ Docker build success

### Deploy to PROD
1. ✅ Smoke E2E tests na STAGE zielone
2. ✅ Manual QA approval
3. ✅ No open CRITICAL bugs

---

## 10. Chaos Engineering (TST-01)

> **Cel:** Regularnie weryfikować odporność systemu na awarie w kontrolowanych warunkach.

### 10.1 Scenariusze

| # | Scenariusz | Metoda | Oczekiwany wynik | Częstotliwość |
|---|---|---|---|---|
| CH-01 | Redis down | `docker compose stop redis` | Panel states = UNKNOWN w < 5 min, alert MON-04, snapshot fallback | Miesięcznie |
| CH-02 | RabbitMQ down | `docker compose stop rabbitmq` | Outbox gromadzi eventy w PostgreSQL, po restart relay dostarcza backlog | Miesięcznie |
| CH-03 | PostgreSQL primary failover | `docker compose stop pg-primary` | Patroni promuje replikę < 30s, PgBouncer przekierowuje, brak utraty danych | Kwartalnie |
| CH-04 | Backend-1 crash | `docker compose kill backend-1` | Nginx przełącza ruch na backend-2, WS reconnect < 5s, brak utraconych alarmów | Miesięcznie |
| CH-05 | Satel Worker crash | `docker compose restart satel-worker` | Leader election reassign < 60s, Full State Dump, delta events | Miesięcznie |
| CH-06 | Network partition (Worker ↔ ETHM-1) | `iptables DROP` na satel_vlan | Worker wykrywa disconnect < 3s, reconnect z backoff, alert SatelWorkerDown | Kwartalnie |
| CH-07 | Disk full (RabbitMQ) | Fill /var/lib/rabbitmq to 95% | RabbitMQ disk alarm, alert, runbook RUN-017 triggered | Kwartalnie |
| CH-08 | Burst 500 alarmów | Symulator: 500 zone violations w 10s | Kolejka rośnie, brak utraconych eventów, delivery < 5s p95 | Kwartalnie |

### 10.2 Zasady

- **Tylko na STAGE** (nigdy na PROD bez jawnej autoryzacji MASTER).
- Każdy test ma rollback plan (restart kontenera).
- Wyniki dokumentowane w tabeli: data, scenariusz, wynik, odkryte problemy.
- Przy awarii testu → tworzony Action Item wg szablonu postmortem (`18_RUNBOOK.md`).

---

## 11. Disaster Recovery Drills (TST-02)

> **Cel:** Regularnie ćwiczyć procedury DR i mierzyć faktyczny RTO.

### 11.1 Harmonogram

| Typ testu | Częstotliwość | Zakres | Kto prowadzi |
|---|---|---|---|
| **Tabletop exercise** | Co miesiąc | Przejście runbook na sucho (RUN-001 do RUN-018) | ADMIN + Developer |
| **Failover drill** | Co kwartał | Scenariusz CH-03 (Patroni failover) + CH-04 (Backend crash) | DevOps |
| **Full DR drill** | Co pół roku | Restore z backupu na czystym serwerze, zmierz RTO | DevOps + MASTER |
| **Chaos day** | Co pół roku | Losowe CH-01 do CH-08, bez uprzedzenia zespołu | DevOps (oznaczony "game master") |

### 11.2 Metryki sukcesu

| Metryka | Cel |
|---|---|
| Zmierzony RTO (PostgreSQL failover) | < 30s |
| Zmierzony RTO (Full DR restore) | < 2h |
| Procedury wymagające aktualizacji po drilli | Zidentyfikowane i zaktualizowane w ciągu 1 tygodnia |
| Odkryte luki w runbook | 100% zamknięte przed kolejnym drillem |

---

## 12. Data Loss Simulation (TST-03)

> **Cel:** Zweryfikować, że system **nie traci danych** w scenariuszach awarii.

| # | Scenariusz | Weryfikacja | Oczekiwany wynik |
|---|---|---|---|
| DL-01 | Redis restart podczas aktywnego pollingu | Porównaj eventy w RabbitMQ przed/po | Zero utraconych eventów (Worker odbudowuje z DB) |
| DL-02 | Kill Backend w trakcie INSERT alarmu | Sprawdź PostgreSQL + Outbox | Alarm w DB (transakcja committed) LUB brak (rollback) — nigdy partial |
| DL-03 | RabbitMQ restart z pending messages | Sprawdź persistent messages | Zero utraconych (durable queue + persistent delivery) |
| DL-04 | Sieciowy disconnect Worker ↔ ETHM-1 w trakcie state dump | Porównaj Redis state z następnym full dump | Następny dump koryguje, delta events poprawne |
| DL-05 | PostgreSQL primary crash w trakcie batch INSERT | Sprawdź replica po failover | Committed transactions present, uncommitted rolled back |

---

## 13. Race Condition Tests (TST-04)

> **Cel:** Przetestować wyścigi danych, które mogą prowadzić do nieoczekiwanego zachowania.

| # | Scenariusz | Opis | Oczekiwany wynik |
|---|---|---|---|
| RC-01 | Bundle auto-close + re-trigger | Bundle RESOLVED wchodzi w 30 min timeout do CLOSED. W tym samym momencie nowy event re-triggeruje bundle. | Nowy event tworzy NOWY bundle (nie wchodzi w zamykany). `auto_close_task` ignoruje jeśli `status != RESOLVED`. |
| RC-02 | Concurrent CLAIM | 2 operatorów klikają CLAIM na ten sam bundle w < 1s | Tylko 1 wygrywa (optimistic lock `version`). Drugi dostaje `409 STALE_VERSION`. |
| RC-03 | Worker poll + command send | Worker wysyła komendę ARM i jednocześnie poll odczytuje stan stref | Command ma wyższy priorytet. Poll czeka na odpowiedź ARM. |
| RC-04 | Outbox relay + DB cleanup | Outbox relay czyta pending entries. Jednocześnie cleanup job usuwa stare entries. | Relay NIGDY nie pominie entry (SELECT ... FOR UPDATE SKIP LOCKED). |
| RC-05 | WS reconnect + event delivery | Reset WebSocket w trakcie dostarczania eventu | `sequence_id` gwarantuje brak duplikatów i brak utraconych (klient wysyła `replay_request`). |

**Implementacja:** Testy używające `asyncio.gather()` do symulacji concurrent requests + `pytest-repeat` z 100 powtórzeń per scenariusz.
