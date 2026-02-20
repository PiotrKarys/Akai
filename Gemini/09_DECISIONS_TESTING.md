# 12_DECISIONS_AND_OPEN

## Cel
Rejestr podjętych decyzji architektonicznych (ADR) oraz lista otwartych pytań.

---

## 1. Podjęte Decyzje (ADR)

### ADR-001: Technologia Frontend
- **Decyzja:** Flutter.
- **Uzasadnienie:** Jeden kod na Windows Desktop (Operator) i Android (Technik). Wsparcie offline.

### ADR-002: Baza Danych
- **Decyzja:** PostgreSQL.
- **Uzasadnienie:** Standard przemysłowy, niezawodność, łatwość replikacji, obsługa JSONB (dla metadanych sensorów).

### ADR-003: Zarządzanie Sekretami (MVP)
- **Decyzja:** Pliki `.env` / Docker Secrets.
- **Status:** Zaakceptowane dla fazy początkowej. Vault to overhead na start.

### ADR-004: Integracja SMS (MVP)
- **Decyzja:** Modem fizyczny + Symulacja.
- **Status:** Brak integracji z zewnętrznymi bramkami API w pierwszej fazie.

### ADR-005: Storage Plików
- **Decyzja:** Lokalny system plików serwera.
- **Status:** Prosty wolumen Dockera. Migracja na S3/MinIO w wersji Enterprise (v2.0).

### ADR-006: Integracja z Centralami SATEL
- **Decyzja:** Bezpośrednie połączenie TCP/IP z modułem ETHM-1 Plus (port 10004), oficjalny protokół integracji (ramki binarne 0xFE 0xFE, CRC-16).
- **Uzasadnienie:** Real-time monitoring (<1s reakcji), brak zależności od plików STAM/DLOAD, oficjalna dokumentacja protokołu dostępna.
- **Konsekwencje:** Dedykowany mikroserwis Satel Worker (Python Asyncio), ograniczenie ETHM-1 do 1 połączenia per moduł.
- **Status:** Zaakceptowane. Użytkownik posiada oficjalny dokument "Protokół integracji ETHM-1".

### ADR-007: Live State Cache
- **Decyzja:** Redis.
- **Uzasadnienie:** Przechowywanie aktualnego stanu central (wejścia, strefy, awarie) z dostępem <1ms. Odciąża PostgreSQL od ciągłych UPDATE'ów stanu live. Cache odtwarzalny z DB + fresh poll z central.
- **Konsekwencje:** Dodatkowy kontener w Docker Compose, ale zero ryzyka utraty danych (dane tymczasowe).
- **Status:** Zaakceptowane.

### ADR-008: Read-Only Replica PostgreSQL
- **Decyzja:** Wdrożenie read-only repliki PostgreSQL (Streaming Replication) do separacji ruchu transakcyjnego od analitycznego.
- **Uzasadnienie:** Raporty i eksporty danych mogą przeciążyć primary DB. Replica odciąża główną bazę, zapewniając stabilność operacyjną.
- **Konsekwencje:** Dodatkowy kontener `db-replica`, dwa connection stringi w backendzie (`DATABASE_URL` + `DATABASE_REPLICA_URL`).
- **Status:** Zaakceptowane. Szczegóły: **02_ARCHITECTURE.md sekcja 2.6.1**, **11_DEPLOYMENT.md sekcja 7**.

### ADR-009: Priorytetowe Kolejki Komend RabbitMQ
- **Decyzja:** Rozdzielenie kolejki komend na `commands.satel.high` (Arm/Disarm/ClearAlarm) i `commands.satel.low` (Output ON/OFF).
- **Uzasadnienie:** Komendy krytyczne (Arm/Disarm) muszą być realizowane natychmiast, nawet przy dużym obciążeniu systemu.
- **Konsekwencje:** Satel Worker konsumuje z dwóch kolejek, priorytetowo obsługując `high`.
- **Status:** Zaakceptowane. Szczegóły: **13_EVENT_SCHEMAS.md sekcja 1**.

### ADR-010: Harmonogram Automatycznego Uzbrajania (Auto-Arm)
- **Decyzja:** Dedykowany cron job w backendzie wysyła komendy Arm/Disarm wg harmonogramu per obiekt.
- **Uzasadnienie:** Obiekty typu przychodnie wymagają automatycznego uzbrajania o stałej porze (np. 24:00). Zmniejsza ryzyko ludzkiego błędu.
- **Konsekwencje:** Nowy moduł `Auto-Arm Scheduler`, logowanie w AUDIT_LOG z `source: "AUTO_ARM"`, obsługa błędów (alarm WARNING przy braku łączności).
- **Status:** Zaakceptowane. Szczegóły: **03_FUNCTIONAL_MODULES.md sekcja 16**.

### ADR-011: Optimistic Locking (v7.0 Remediation)
- **Decyzja:** Dodanie kolumny `version` (INTEGER, default 1) do tabel `BUNDLE_ALARMS`, `OBJECTS`, `PANELS`.
- **Uzasadnienie:** Zapobieganie nadpisywaniu równoczesnych edycji (race condition RC-001). Każda mutacja wymaga podania aktualnej `version` — niezgodność = `409 ALARM_STALE_VERSION`.
- **Alternatywy odrzucone:** PostgreSQL `xmin` (niestabilny przy vacuum), `SELECT FOR UPDATE` (blokujący, nie skaluje).
- **Konsekwencje:** Frontend musi przechowywać `version` i wysyłać go w każdym PATCH/POST.
- **Status:** Zaakceptowane. Szczegóły: **04_DATA_MODEL_ERD.md**, **05_ALARM_LIFECYCLE.md (tabela przejść)**, **10_API_HIGH_LEVEL.md**.

### ADR-012: Outbox Pattern (v7.0 Remediation)
- **Decyzja:** Tabela `outbox` w PostgreSQL jako bufor między aplikacją a RabbitMQ. Dedykowany relay (co 100ms) publikuje wiadomości.
- **Uzasadnienie:** Eliminacja dual-write problem (DB zapisane, RabbitMQ nie). Gwarantuje atomowość: event jest zawsze w DB zanim pojawi się w kolejce.
- **Alternatywy odrzucone:** Transactional RabbitMQ Publisher Confirms (nie gwarantują atomowości z DB commit), CDC/Debezium (zbyt ciężkie na MVP).
- **Konsekwencje:** ~100ms dodatkowej latencji na publish. Nowa tabela + background task. Relay musi być idempotentny.
- **Status:** Zaakceptowane. Szczegóły: **04_DATA_MODEL_ERD.md sekcja 7a**, **13_EVENT_SCHEMAS.md sekcja 7**.

### ADR-013: Pending States — CLAIMING, ARMING/DISARMING (v7.0 Remediation)
- **Decyzja:** Dodanie stanów przejściowych: `CLAIMING` (DB-persisted) w lifecycle alarmu + `ARMING_STAY`, `ARMING_AWAY`, `DISARMING` (Redis-only) w live state partycji.
- **Uzasadnienie:** Operator musi widzieć feedback natychmiast po kliknięciu (UX). Eliminacja "kliknąłem i nic się nie dzieje przez 2s".
- **CLAIMING (DB):** Max 5s TTL, revert do NEW przy timeout/błędzie. Zapobiega double-claim.
- **ARMING/DISARMING (Redis):** Sub-second, Worker-owned. Nie persystowane w PostgreSQL — to cache UI.
- **Konsekwencje:** Frontend musi obsłużyć stany przejściowe (spinner, disabled button). Satel Commands tracking w tabeli `satel_commands`.
- **Status:** Zaakceptowane. Szczegóły: **05_ALARM_LIFECYCLE.md**, **13_EVENT_SCHEMAS.md sekcja 5**.

### ADR-014: WebSocket Catch-up Mechanism — Tiered (v7.0 Remediation)
- **Decyzja:** Każda wiadomość WebSocket zawiera `sequence_id` (monotonicznie rosnący). **Tier 1:** Redis Sorted Set `ws:events:buffer` przechowuje ostatnie **5000** eventów. **Tier 2:** Jeśli `last_sequence_id` starszy niż Redis buffer (ale <2h), fallback na PostgreSQL (tabela `outbox`, `sequence_id`). **Tier 3:** >2h → `replay_overflow` z `retry_after_ms` (ochrona thundering herd).
- **Uzasadnienie:** WebSocket nie gwarantuje dostarczenia. Tiered catch-up zapewnia odporność na restart Redisa i dłuższe rozpołączenia.
- **Alternatywy odrzucone:** SSE (brak dwukierunkowej komunikacji), polling (opóźnienia nieakceptowalne).
- **Status:** Zaakceptowane. Szczegóły: **10_API_HIGH_LEVEL.md sekcja 8.5**, **02_ARCHITECTURE.md sekcja 2.6**.

### ADR-015: Auto-Arm Conflict Resolution — Manual > Automatic (v7.0 Remediation)
- **Decyzja:** Manualna komenda operatora ZAWSZE wygrywa z automatycznym harmonogramem. Po manualnym DISARM, Auto-Arm pomija obiekt przez `AUTO_ARM_COOLDOWN_MINUTES` (domyślnie 60 min).
- **Uzasadnienie:** Eliminacja race condition RC-005. Bezpieczeństwo fizyczne > automatyzacja.
- **Łańcuch decyzyjny:** **6 kroków**: mode check → cooldown check → **DLOAD service window check** → in-flight check → connection check → execute.
- **Konsekwencje:** Cooldown konfigurowalny per deployment. AUDIT_LOG zapisuje zarówno sukces jak i SKIP z powodem.
- **Status:** Zaakceptowane. Szczegóły: **03_FUNCTIONAL_MODULES.md sekcja 16**.

### ADR-016: Offline Intent Queue (v8.0 Remediation)
- **Decyzja:** Tryb offline = read-only. Przyciski mutacji wyłączone. Operator może zapisywać lekkie intencje (INTENT_CLAIM, INTENT_NOTE, INTENT_SERVICE_TICKET) synchonizowane po reconnect przez `POST /api/sync/intents`.
- **Uzasadnienie:** Full mutation queue offline generuje masywne konflikty 409 przy sync (stale `version`). Intent Queue eliminuje ten problem — backend arbitruje z bieżącym stanem.
- **Status:** Zaakceptowane. Szczegóły: **03_FUNCTIONAL_MODULES.md sekcja 19.4**, **10_API_HIGH_LEVEL.md sekcja 5b**.

### ADR-017: PII Structural Isolation (v8.0 Remediation)
- **Decyzja:** Pole `raw_sms` zastąpione przez `raw_sms_hash` (SHA-256) w RabbitMQ i `EVENTS.details`. Oryginalny tekst szyfrowany (Fernet) i przechowywany w osobnej tabeli `sms_raw_archive` z ograniczonym dostępem (MASTER/SYSTEM).
- **Uzasadnienie:** PII nie powinno być publikowane do message bus ani przechowywane w głównych tabelach dostępnych wwide. Strukturalna izolacja > policy-based masking.
- **Status:** Zaakceptowane. Szczegóły: **13_EVENT_SCHEMAS.md sekcja 3**, **08_SECURITY_AND_ROLES.md**, **04_DATA_MODEL_ERD.md sekcja 12**.

### ADR-018: Cursor Pagination for Alarms (v8.0 Remediation)
- **Decyzja:** Endpoint `GET /api/alarms` używa Keyset (Cursor) Pagination zamiast offset-based. Inne endpointy ze statycznymi danymi zachowują offset pagination.
- **Uzasadnienie:** Na dynamicznie zmieniającej się liście alarmów, offset pagination powoduje duplikaty i pominięcia (nowe alarmy przesuwają stare).
- **Status:** Zaakceptowane. Szczegóły: **10_API_HIGH_LEVEL.md sekcja 9**.

### ADR-019: No STAM Migration — Excel Import Only
- **Decyzja:** Brak automatycznej migracji z bazy STAM (Firebird/Interbase). Dane obiektów importowane ręcznie przez Excel (`POST /api/admin/import/objects`).
- **Uzasadnienie:** Baza STAM nie jest dostępna programowo. Struktury tabel są nieznane. Ręczny import jest kontrolowany i audytowalny.
- **Konsekwencje:** Nowy plik `19_DEV_TOOLING.md` z definicją schematu Excel. Tabela `IMPORT_LOG` w ERD. Endpoint `POST /api/admin/import/objects`.
- **Status:** Zaakceptowane. Szczegóły: **19_DEV_TOOLING.md sekcja 4**, **10_API_HIGH_LEVEL.md sekcja 13**.

---

## 2. Otwarte Pytania (Do rozstrzygnięcia w trakcie)

1. **Mapy Offline:** Czy na mobile potrzebujemy pełnych map offline (duży rozmiar), czy wystarczy cache kafelków Google/OSM z ostatnich wizyt?
2. ~~**Migracja danych:** Czy mamy dostęp do bazy STAM (Firebird/Interbase) żeby zmigrować historię, czy startujemy z czystą kartą?~~ **✅ RESOLVED: Brak migracji STAM.** Startujemy z czystą kartą + import Excel. Szczegóły: ADR-019.
3. ~~**DB failover tooling:** Patroni / pg_auto_failover / repmgr — do decyzji przy wdrożeniu HA (v1.0).~~ **✅ RESOLVED: Patroni** (automatyczny failover, etcd consensus, PgBouncer connection pooling). Szczegóły: `09_HA_RTO_RPO.md` (HA-01), `11_DEPLOYMENT.md`.


---
---


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


---
---


# 19_DEV_TOOLING

## Cel
Narzędzia deweloperskie, konwencje logowania, feature flags i strategie danych testowych. Obowiązuje od **Fazy 1**.

---

## 1. Seed Data Strategy

### Cel
Automatyczne wypełnienie bazy danych w środowisku DEV danymi testowymi. Skrypt uruchamiany przy `docker compose up` (profil DEV).

### Skrypt: `seed_dev.py`

**Lokalizacja:** `backend/scripts/seed_dev.py`

**Uruchomienie:** Automatycznie przy starcie kontenera backend w DEV (`entrypoint.sh` → `python scripts/seed_dev.py`).

**Idempotencja:** Skrypt MUSI być idempotentny — wielokrotne uruchomienie nie tworzy duplikatów. Sprawdza `EXISTS` przed INSERT.

### Dane testowe

| Encja | Ilość | Opis |
|---|---|---|
| **OBJECTS** | 10 | Różne typy (biuro, magazyn, przychodnia, dom, szkoła) |
| **PANELS** | 3 per obiekt (30 total) | Panel SATEL z konfiguracją ETHM-1 |
| **USERS** | 3 per rola (18 total) | Po 3 użytkowników na każdą z 6 ról |
| **BUNDLE_ALARMS** (historia) | 50 | Różne statusy (NEW, IN_PROGRESS, ACK, RESOLVED, CLOSED) |
| **SMS_PHONE_MAPPING** | 5 | Zaufane numery Efento/Bluelog |
| **ZONES** | 5 per panel (150 total) | Strefy z `priority_override` |
| **OBJECT_CONTACTS** | 2 per obiekt (20 total) | Kontakty alarmowe |
| **OBJECT_PROCEDURES** | 1 per obiekt (10 total) | Procedury postępowania |

### Spójność z testami

- **pytest (backend):** Fabryki testowe (`conftest.py`) MUSZĄ używać tych samych danych co seed.
- **Flutter (frontend):** Mock providers MUSZĄ zwracać dane zgodne ze seed.
- **Źródło prawdy:** Definicja danych testowych w pliku `backend/tests/factories/seed_data.py` — importowana zarówno przez `seed_dev.py` jak i fabryki pytest.

---

## 2. Logowanie Strukturalne

### Wymaganie
Logowanie strukturalne obowiązuje od **Fazy 1** we wszystkich serwisach.

### Biblioteka
`structlog` (Python) — dodać do `requirements.txt` / `pyproject.toml`.

### Format logów

```json
{
  "timestamp": "2026-02-20T14:30:00.123Z",
  "level": "INFO",
  "service": "backend",
  "event": "Alarm created for panel_001",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "context": {
    "panel_id": "panel_001",
    "event_type": "ALARM",
    "user_id": "uuid-of-operator"
  }
}
```

### Reguły

| Reguła | Opis |
|---|---|
| **request_id** | UUID generowany per request. MUSI być obecny w **każdym** loginie. Propagowany w headerze `X-Request-ID`. |
| **Serwisy** | `backend`, `satel-worker`, `sms-agent`, `outbox-relay` |
| **Poziomy** | `DEBUG` (tylko DEV), `INFO`, `WARNING`, `ERROR`, `CRITICAL` |
| **Korelacja** | `request_id` pozwala śledzić request przez cały pipeline (Nginx → Backend → RabbitMQ → Worker) |
| **Nie loguj PII** | Nigdy nie loguj treści SMS, haseł, tokenów JWT. Dozwolone: `user_id`, `panel_id`, `object_id`. |

### Konfiguracja structlog

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
)
```

### Powiązania
- Retencja logów: **17_MONITORING.md, sekcja 2**
- Poziomy logowania: **17_MONITORING.md, sekcja 2**

---

## 3. Feature Flags

### Cel
Kontrola dostępności funkcji między fazami. Pozwalają na bezpieczny deploy kodu Fazy N+1 bez aktywowania nowych funkcji.

### Plik konfiguracyjny: `feature_flags.yaml`

**Lokalizacja:** `backend/config/feature_flags.yaml`

```yaml
# Feature Flags — kontrola dostępności funkcji per faza
# Zmiana flagi wymaga restartu serwisu (reload config).

features:
  FEATURE_REALTIME_ENABLED: false    # Faza 1: false → Faza 2: true
  FEATURE_PUSH_ENABLED: false        # Faza 1: false → Faza 2: true
  FEATURE_SMS_ENABLED: false         # Faza 1: false → Faza 4: true
  FEATURE_ARM_DISARM_ENABLED: false  # Faza 1: false → Faza 5: true
  FEATURE_HA_ENABLED: false          # Faza 1: false → Faza 6: true
```

### Tabela flag

| Flaga | Faza OFF | Faza ON | Co kontroluje |
|---|---|---|---|
| `FEATURE_REALTIME_ENABLED` | 1 | **2** | RabbitMQ, Redis, WebSocket, Outbox Relay, Satel Worker |
| `FEATURE_PUSH_ENABLED` | 1 | **2** | Firebase Cloud Messaging (FCM), tabela `USER_DEVICES` |
| `FEATURE_SMS_ENABLED` | 1 | **4** | SMS Agent, parser Efento/Bluelog, modem GSM |
| `FEATURE_ARM_DISARM_ENABLED` | 1 | **5** | Komendy sterujące ARM/DISARM, priority queues |
| `FEATURE_HA_ENABLED` | 1 | **6** | Patroni, etcd, PgBouncer, Redis Sentinel, multi-instance backend |

### Logika backendu

```python
# Warunkowa inicjalizacja połączeń
if settings.FEATURE_REALTIME_ENABLED:
    await connect_redis()
    await connect_rabbitmq()
    register_websocket_routes(app)
    start_outbox_relay()

if settings.FEATURE_PUSH_ENABLED:
    initialize_fcm_client()

if settings.FEATURE_SMS_ENABLED:
    start_sms_consumer()
```

### Wpływ na `/readyz`

Endpoint `GET /readyz` sprawdza **tylko włączone** zależności:

```python
checks = {"postgresql": check_pg()}

if settings.FEATURE_REALTIME_ENABLED:
    checks["redis"] = check_redis()
    checks["rabbitmq"] = check_rabbitmq()

return {"status": "ready", "checks": checks}
```

> [!WARNING]
> **Zmiana flagi wymaga restartu serwisu.** Nie stosujemy hot-reload flag — upraszcza to logikę i eliminuje ryzyko niespójnego stanu.

### Powiązania
- Architektura: **02_ARCHITECTURE.md** — feature flags jako mechanizm przełączania faz
- Health checks: **09_HA_RTO_RPO.md, sekcja HA-03** — `/readyz` sprawdza tylko włączone zależności

---

## 4. Schemat Importu Excel

### Cel
Definicja formatu pliku `.xlsx` akceptowanego przez endpoint `POST /api/admin/import/objects`.

### Wymagane kolumny

| Kolumna | Typ | Wymagana | Opis |
|---|---|---|---|
| `nazwa` | string | ✅ | Unikalna nazwa obiektu |
| `adres` | string | ✅ | Adres obiektu (geokodowanie automatyczne) |
| `typ` | string | ✅ | Typ obiektu (biuro, magazyn, przychodnia, dom, szkoła, inny) |
| `kontakt_telefon` | string | ❌ | Numer telefonu kontaktowego |
| `kontakt_email` | string | ❌ | Email kontaktowy |
| `kontakt_imie` | string | ❌ | Imię i nazwisko osoby kontaktowej |
| `notatki` | string | ❌ | Dodatkowe informacje |

### Walidacja

| Reguła | Opis | Kod błędu |
|---|---|---|
| Unikalność nazwy | Nazwa obiektu musi być unikalna w systemie | `IMPORT_DUPLICATE_NAME` |
| Format pliku | Tylko `.xlsx` (nie `.xls`, nie `.csv`) | `IMPORT_INVALID_FORMAT` |
| Wymagane kolumny | Brak kolumny `nazwa`, `adres` lub `typ` → błąd | `IMPORT_INVALID_FORMAT` |
| Limit wierszy | Max 1000 wierszy per import | `IMPORT_INVALID_FORMAT` |

### Powiązania
- Endpoint API: **10_API_HIGH_LEVEL.md** — `POST /api/admin/import/objects`
- Tabela logów: **04_DATA_MODEL_ERD.md** — `IMPORT_LOG`
- ADR: **12_DECISIONS_AND_OPEN.md** — ADR-019
