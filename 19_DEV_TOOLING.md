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
