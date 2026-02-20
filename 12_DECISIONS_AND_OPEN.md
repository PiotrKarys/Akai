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

---

## 2. Otwarte Pytania (Do rozstrzygnięcia w trakcie)

1. **Mapy Offline:** Czy na mobile potrzebujemy pełnych map offline (duży rozmiar), czy wystarczy cache kafelków Google/OSM z ostatnich wizyt?
2. **Migracja danych:** Czy mamy dostęp do bazy STAM (Firebird/Interbase) żeby zmigrować historię, czy startujemy z czystą kartą?
3. ~~**DB failover tooling:** Patroni / pg_auto_failover / repmgr — do decyzji przy wdrożeniu HA (v1.0).~~ **✅ RESOLVED: Patroni** (automatyczny failover, etcd consensus, PgBouncer connection pooling). Szczegóły: `09_HA_RTO_RPO.md` (HA-01), `11_DEPLOYMENT.md`.
