# 13_EVENT_SCHEMAS

## Cel
Definicja formatów wiadomości (event schemas) przesyłanych przez RabbitMQ. Ten dokument jest **kontraktem** między Satel Worker, SMS Agent i Backend API.

> [!IMPORTANT]
> **Mandat UTC (SIL-07):** Wszystkie timestampy w event schemas MUSZĄ być w formacie **ISO 8601 z timezone UTC** (suffix `Z`). Konwersja na czas lokalny odbywa się TYLKO w warstwie UI (Flutter). Backend, Worker, SMS Agent — wszystko operuje na UTC. Patrz również: `09_HA_RTO_RPO.md`, sekcja NTP.

---

## 1. Topologia RabbitMQ

### Exchanges

| Exchange | Typ | Opis |
|---|---|---|
| `satel.events` | topic | Zdarzenia z central alarmowych |
| `sms.events` | topic | Zdarzenia z parserów SMS |
| `system.commands` | direct | Komendy sterujące do Worker (v2.0) |
| `system.dlx` | fanout | Dead Letter Exchange |

### Queues

| Queue | Bound to | Routing Key | Konsument |
|---|---|---|---|
| `events.processing` | satel.events | `satel.#` | Backend API |
| `events.processing` | sms.events | `sms.#` | Backend API |
| `commands.satel.high` | system.commands | `cmd.satel.high` | Satel Worker (priorytet) |
| `commands.satel.low` | system.commands | `cmd.satel.low` | Satel Worker |
| `events.dead` | system.dlx | `#` | Monitoring / Manual Review |

### Priorytetyzacja komend

| Kolejka | Komendy | Opis |
|---|---|---|
| `commands.satel.high` | `ARM_STAY`, `ARM_AWAY`, `DISARM`, `CLEAR_ALARM` | Komendy krytyczne — wymagają natychmiastowej realizacji |
| `commands.satel.low` | `OUTPUT_ON`, `OUTPUT_OFF`, inne | Komendy niekrytyczne — mogą poczekać |

**Mechanizm:**
- Backend publikuje komendy do odpowiedniej kolejki na podstawie `command_type`.
- Satel Worker konsumuje z obu kolejek, ale priorytetowo obsługuje `commands.satel.high`:
  - Osobny konsumer/kanał dla kolejki `high` z wyższym prefetch (prefetch: 5).
  - Kolejka `low` z prefetch: 1.
  - Worker sprawdza `high` przed `low` w każdym cyklu.
- Priorytet: v1.0 (zapewnia szybką reakcję na komendy krytyczne nawet przy dużym obciążeniu).

### Parametry kolejek

| Parametr | Wartość |
|---|---|
| Durable | true |
| Auto-delete | false |
| Message TTL | 24h (86400000 ms) |
| Max retries | 3 (potem → DLQ) |
| Prefetch count | 10 (Backend), 1 (Worker) |

---

## 2. Event Schema: Satel Alarm Event

**Producent:** Satel Worker
**Exchange:** `satel.events`
**Routing Key:** `satel.alarm.{panel_id}` lub `satel.status.{panel_id}`

### Struktura wiadomości

| Pole | Typ | Wymagane | Opis |
|---|---|---|---|
| `event_id` | string (UUID v4) | ✓ | Unikalny identyfikator zdarzenia |
| `timestamp` | string (ISO 8601) | ✓ | Czas zdarzenia z centrali |
| `received_at` | string (ISO 8601) | ✓ | Czas odebrania przez Worker |
| `source` | enum | ✓ | Stałe: `"SATEL"` |
| `panel_id` | string | ✓ | ID centrali w systemie |
| `event_type` | enum | ✓ | `ALARM` / `TAMPER` / `INFO` / `ARM` / `DISARM` / `TROUBLE` / `UNAUTHORIZED_ACCESS` |
| `zone_id` | string | nullable | ID strefy / wejścia (null dla zdarzeń systemowych) |
| `partition_id` | string | nullable | ID partycji (null dla zdarzeń wejść) |
| `event_code` | string | ✓ | Kod zdarzenia hex z protokołu ETHM-1 |
| `description` | string | ✓ | Opis czytelny ("Włamanie strefa 1", "Awaria zasilania") |
| `priority` | enum | ✓ | `CRITICAL` / `WARNING` / `INFO` — **Uwaga:** ustawiane przez Backend po enrichment (patrz niżej) |
| `default_priority` | enum | ✓ | `CRITICAL` / `WARNING` / `INFO` — priorytet domyślny przypisany przez Worker na podstawie `event_code` |
| `raw_payload` | string (hex) | ✓ | Surowa ramka binarna (do debugowania) |
| `dedup_key` | string | ✓ | Klucz deduplikacji: `{panel_id}:{event_code}:{zone_id}:{timestamp_minute}` |
| `service_instance_id` | string | ✓ | Identyfikator instancji Worker (np. `satel-worker-1`). Cel: traceability w środowisku multi-instance. |

> [!IMPORTANT]
> **Two-Phase Priority Enrichment:** Worker ustawia `default_priority` na podstawie kodu zdarzenia sprzętowego (brak kontekstu biznesowego). Backend wzbogaca końcowe `priority` konsultując `ZONES.priority_override` (JSONB) — jeśli override istnieje dla danego `event_type` i `zone_id`, używany jest override. W przeciwnym razie `priority = default_priority`. Override konfigurowany przez ADMIN/MASTER w ustawieniach strefy.

### Przykład wiadomości

```json
{
  "event_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "timestamp": "2026-02-12T14:30:00.123Z",
  "received_at": "2026-02-12T14:30:00.456Z",
  "source": "SATEL",
  "panel_id": "panel_001",
  "event_type": "ALARM",
  "zone_id": "zone_003",
  "partition_id": "part_01",
  "event_code": "0x01",
  "description": "Włamanie — strefa 1, wejście 3",
  "priority": "CRITICAL",
  "raw_payload": "FEFE0E01030000...CRC",
  "dedup_key": "panel_001:0x01:zone_003:202602121430",
  "service_instance_id": "satel-worker-1"
}
```

---

## 3. Event Schema: SMS Temperature Event

**Producent:** SMS Agent
**Exchange:** `sms.events`
**Routing Key:** `sms.temp.{source_type}` (np. `sms.temp.efento`, `sms.temp.bluelog`)

### Struktura wiadomości

| Pole | Typ | Wymagane | Opis |
|---|---|---|---|
| `event_id` | string (UUID v4) | ✓ | Unikalny identyfikator |
| `timestamp` | string (ISO 8601) | ✓ | Timestamp z treści SMS (parsowany) |
| `received_at` | string (ISO 8601) | ✓ | Czas odebrania SMS przez modem |
| `source` | enum | ✓ | `"SMS_EFENTO"` / `"SMS_BLUELOG"` |
| `sender_phone` | string | ✓ | Numer nadawcy (np. "+48500123456") |
| `event_type` | enum | ✓ | `TEMP_ALARM` / `TEMP_NORMAL` |
| `sensor_name` | string | nullable | Nazwa czujnika (Efento: np. "Leg_szczep_prawa") |
| `device_id` | string | nullable | ID urządzenia (Bluelog: np. "21040DD5") |
| `channel` | string | nullable | Kanał (Bluelog: np. "S1") |
| `location_label` | string | ✓ | Lokalizacja z SMS ("Gad Spedycja") |
| `temperature_value` | float | nullable | Zmierzona wartość (np. 1.7) |
| `temperature_unit` | string | ✓ | `"C"` (Celsius) |
| `threshold_value` | float | nullable | Próg alarmowy (Bluelog: np. 15.5) |
| `rule_name` | string | nullable | Nazwa reguły (Efento: np. "Leg_szczep_prawa_MIN") |
| `priority` | enum | ✓ | `CRITICAL` (alarm) / `INFO` (powrót do normy) |
| `sms_quality` | enum | ✓ | `complete` / `truncated` / `garbled` / `unparseable` — szczegóły: `06_INTEGRATIONS.md`, sekcja 2.3a |
| `raw_sms_hash` | string | ✓ | SHA-256 hash pełnej treści SMS. **Oryginalny tekst przechowywany w osobnej tabeli `sms_raw_archive`** (dostęp: MASTER/SYSTEM). W logach produkcyjnych i w RabbitMQ/PostgreSQL NIGDY nie występuje pełna treść SMS. |
| `dedup_key` | string | ✓ | `{source}:{sensor_name/device_id}:{event_type}:{timestamp_minute}` |
| `service_instance_id` | string | ✓ | Identyfikator instancji SMS Agent (np. `sms-agent-1`) |

### Przykład: Efento Alarm

```json
{
  "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2026-02-10T11:03:00.000Z",
  "received_at": "2026-02-10T11:03:12.000Z",
  "source": "SMS_EFENTO",
  "sender_phone": "+48500111222",
  "event_type": "TEMP_ALARM",
  "sensor_name": "Leg_szczep_prawa",
  "device_id": null,
  "channel": null,
  "location_label": "Swiat Zdrowia Operat - Leg_Szczep",
  "temperature_value": 1.7,
  "temperature_unit": "C",
  "threshold_value": null,
  "rule_name": "Leg_szczep_prawa_MIN",
  "priority": "CRITICAL",
  "sms_quality": "complete",
  "raw_sms_hash": "a3f2b8c1d4e5f6a7b8c9d0e1f2345678901234567890abcdef1234567890abcd",
  "dedup_key": "SMS_EFENTO:Leg_szczep_prawa:TEMP_ALARM:202602101103",
  "service_instance_id": "sms-agent-1"
}
```

### Przykład: Bluelog Alert

```json
{
  "event_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "timestamp": "2026-02-10T11:03:00.000Z",
  "received_at": "2026-02-10T11:03:15.000Z",
  "source": "SMS_BLUELOG",
  "sender_phone": "+48500333444",
  "event_type": "TEMP_ALARM",
  "sensor_name": null,
  "device_id": "21040DD5",
  "channel": "S1",
  "location_label": "Gad Spedycja",
  "temperature_value": 15.3,
  "temperature_unit": "C",
  "threshold_value": 15.5,
  "rule_name": null,
  "priority": "CRITICAL",
  "sms_quality": "complete",
  "raw_sms_hash": "b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4",
  "dedup_key": "SMS_BLUELOG:21040DD5:TEMP_ALARM:202602101103",
  "service_instance_id": "sms-agent-1"
}
```

---

## 4. Command Schema: Satel Command (v2.0)

**Producent:** Backend API
**Exchange:** `system.commands`
**Routing Key:** `cmd.satel.high` (Arm/Disarm/ClearAlarm) lub `cmd.satel.low` (Output ON/OFF)

### Struktura wiadomości

| Pole | Typ | Wymagane | Opis |
|---|---|---|---|
| `command_id` | string (UUID v4) | ✓ | Unikalny identyfikator komendy |
| `timestamp` | string (ISO 8601) | ✓ | Czas wydania komendy |
| `issued_by` | string | ✓ | User ID operatora |
| `panel_id` | string | ✓ | ID centrali docelowej |
| `command_type` | enum | ✓ | `ARM_STAY` / `ARM_AWAY` / `DISARM` / `OUTPUT_ON` / `OUTPUT_OFF` |
| `partition_id` | string | nullable | ID partycji (dla ARM/DISARM) |
| `output_id` | string | nullable | ID wyjścia (dla OUTPUT_ON/OFF) |
| `user_code` | string (encrypted) | ✓ | Kod użytkownika centrali (zaszyfrowany) |
| `timeout_ms` | int | ✓ | Timeout oczekiwania na ACK (domyślnie 5000) |

### Odpowiedź (przez osobny routing key `cmd.satel.response`)

| Pole | Typ | Opis |
|---|---|---|
| `command_id` | string | Powiązanie z żądaniem |
| `status` | enum | `ACK` / `NACK` / `TIMEOUT` |
| `error_message` | string | Opis błędu (jeśli NACK) |
| `executed_at` | string | Timestamp wykonania |

---

## 5. Status Update Schema: Redis Live State

**Producent:** Satel Worker (zapisuje bezpośrednio do Redis)
**Konsument:** Backend API (czyta z Redis)

### Klucze Redis

| Klucz | Typ Redis | TTL | Zawartość |
|---|---|---|---|
| `panel:{panel_id}:zones` | Hash | 5 min | zone_id → stan ("OK" / "ALARM" / "TAMPER" / "TROUBLE") |
| `panel:{panel_id}:partitions` | Hash | 5 min | partition_id → stan ("ARMED_STAY" / "ARMED_AWAY" / "DISARMED" / "ALARM" / "EXIT_DELAY" / "ARMING_STAY" / "ARMING_AWAY" / "DISARMING" / "UNKNOWN") |
| `panel:{panel_id}:outputs` | Hash | 5 min | output_id → stan ("ON" / "OFF") |
| `panel:{panel_id}:troubles` | Hash | 5 min | trouble_code → opis ("AC_LOSS" / "BATTERY_LOW" / "TAMPER") |
| `panel:{panel_id}:connection` | String | 2 min | Status połączenia TCP ("CONNECTED" / "RECONNECTING" / "DISCONNECTED") |
| `panel:{panel_id}:last_poll` | String | 5 min | ISO 8601 timestamp ostatniego poll |

### Zasady TTL
- Worker odświeża dane co 1-2s (polling).
- Jeśli TTL wygaśnie (5 min) → znaczy brak kontaktu z centralą → Backend traktuje jako "UNKNOWN".
- `connection` ma krótki TTL (2 min) — jeśli Worker padnie, status szybko przejdzie w "UNKNOWN".

### Stany przejściowe partycji (ARMING/DISARMING)
- **Źródło:** Backend ustawia w Redis gdy `satel_commands.status = SENT`.
- **Czas trwania:** Do momentu ACK/NACK/TIMEOUT od Workera (standard: 5s timeout).
- **Revert:** Na `NACK` lub `TIMEOUT` → Redis wraca do ostatniego stanu z poll Workera.
- **Cel:** Operator widzi w UI spinner z informacją "Uzbrajanie..." / "Rozbrajanie..." zamiast starego stanu.
- **Source of Truth:** Tabela `satel_commands` w PostgreSQL jest autorytatywna. Redis jest cache wyświetlanym w UI.

---

## 6. Deduplikacja

### Algorytm
1. Producent generuje `dedup_key` wg wzoru dla danego źródła.
2. Backend sprawdza `dedup_key` w tabeli EVENTS (kolumna `event_id_unique`).
3. Jeśli klucz istnieje → event ignorowany (log + counter metric).
4. Jeśli nie istnieje → insertujemy do EVENTS i procesujemy bundling.

### Wzory dedup_key

| Źródło | Wzór | Przykład |
|---|---|---|
| SATEL | `{panel_id}:{event_code}:{zone_id}:{timestamp_minute}` | `panel_001:0x01:zone_003:202602121430` |
| SMS_EFENTO | `SMS_EFENTO:{sensor_name}:{event_type}:{timestamp_minute}` | `SMS_EFENTO:Leg_szczep_prawa:TEMP_ALARM:202602101103` |
| SMS_BLUELOG | `SMS_BLUELOG:{device_id}:{event_type}:{timestamp_minute}` | `SMS_BLUELOG:21040DD5:TEMP_NORMAL:202602101103` |

> [!IMPORTANT]
> **Pole `event_type` w kluczu SMS** (`TEMP_ALARM` / `TEMP_NORMAL`) zapobiega fałszywej deduplikacji. Bez niego, jeśli w tej samej minucie nadejdzie SMS "Alarm" i "Powrót do normy" z tego samego czujnika, drugi zostanie błędnie odrzucony jako duplikat.

### Okno deduplikacji
- Granularność: 1 minuta (ten sam event w tej samej minucie = duplikat).
- Konfigurowalne (na wypadek gdyby centrala wysyłała eventy z mniejszymi interwałami).

---

## 7. Outbox Relay

### Opis
Dedykowany background task (asyncio loop) odpowiedzialny za przenoszenie wiadomości z tabeli `outbox` (PostgreSQL) do RabbitMQ. Gwarantuje atomowość zapisu: event jest zawsze w DB zanim pojawi się w kolejce.

### Parametry
| Parametr | Wartość | Opis |
|---|---|---|
| Polling interval | 100ms | Częstotliwość odpytywania tabeli outbox |
| Batch size | 50 | Max wiadomości per iteracja |
| Max retries | 5 | Po przekroczeniu — log CRITICAL, manual review |
| Cleanup retention | 7 dni | Wpisy PUBLISHED usuwane po 7 dniach |
| Locking | `FOR UPDATE SKIP LOCKED` | Bezpieczne dla wielu instancji relay |
| `sequence_id` | BIGINT, monotonic | Globalny monotoniczny identyfikator używany przez WebSocket Tiered Catch-Up (Tier 2: PostgreSQL fallback) |

### Lifecycle wiadomości
```
PENDING → (relay publish) → PUBLISHED → (cleanup job) → ARCHIVED → (cleanup po 7d) → DELETE
```

### Idempotency
- Każda wiadomość ma `idempotency_key` (UNIQUE constraint).
- Format: `{aggregate_type}:{aggregate_id}:{event_type}:{version}`.
- Przykład: `BUNDLE_ALARM:ba_001:alarm.new:1`.
- Jeśli relay opublikuje i crashnie przed UPDATE → następna iteracja ponowi publish. RabbitMQ consumer musi być idempotentny (sprawdzić `dedup_key`).
