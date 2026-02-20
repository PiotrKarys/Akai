# 07_TECH_STACK.md

## Cel
Definicja stosu technologicznego (Tech Stack). Agent AI nie może używać technologii spoza tej listy bez wyraźnej zgody.

---

## Tabela Technologii

| Warstwa | Technologia | Wersja (zalecana) | Uzasadnienie |
|---|---|---|---|
| **Frontend** | Flutter | Stable (Latest) | Jeden kod na Windows Desktop i Android. Dobre wsparcie offline. |
| **Backend** | Python + FastAPI | 3.10+ | Asynchroniczność (ASGI), wydajność, łatwość pisania parserów. |
| **Satel Integration** | Python Asyncio | 3.10+ | Wydajna obsługa wielu połączeń TCP w jednym procesie (lub pool). |
| **Message Queue** | RabbitMQ | Latest | Niezawodność przy "burst" alarmów (200+ w minutę). |
| **Baza Danych** | PostgreSQL | 14+ | Relacyjność, transakcyjność, partycjonowanie (TimeScaleDB opcjonalnie). |
| **State Cache** | **Redis** | Latest | Przechowywanie stanu live (wejścia, awarie, uzbrojenie) przy <1ms dostępu. |
| **ORM** | SQLAlchemy (Async) / Tortoise ORM | Latest | Obsługa async w Pythonie. |
| **Mobile DB** | SQLite | Latest | Lokalny cache na urządzeniu (read-only/sync). |
| **Real-time** | WebSocket | Standard | Push alarmów do frontendu bez opóźnień. |
| **Kontenery** | Docker + Compose | Latest | Łatwość deploymentu dev/prod. |
| **Monitoring** | Prometheus + Grafana | Latest | Metryki systemowe, biznesowe i connection health ETHM-1. |

---

## Biblioteki i Narzędzia (Rekomendowane)

### Backend (Python):
- `pydantic` - walidacja danych.
- `alembic` - migracje bazy danych.
- `aio-pika` - klient RabbitMQ.
- `redis-py` (async) - klient Redis.
- `gammu` / `python-gammu` - obsługa modemu USB.
- `structlog` - logowanie strukturalne (JSON), obowiązuje od Fazy 1. Szczegóły: **19_DEV_TOOLING.md, sekcja 2**.
- *Custom TCP Driver* - implementacja protokołu Satel wg dokumentacji (0xFE 0xFE...).

### Frontend (Flutter):
- `bloc` / `cubit` - zarządzanie stanem.
- `dio` / `http` - komunikacja z API.
- `sqflite` / `drift` - lokalna baza.
- `web_socket_channel` - obsługa WS.

---

## Ograniczenia
1. **Zero Node.js** w backendzie (chyba że do narzędzi pomocniczych CI).
2. **Zero Cloud Dependencies** w core loop - system musi działać 100% offline (bez internetu), tylko w LAN.
3. **Nie używamy Firebase** do powiadomień krytycznych w LAN (tylko jako dodatek do pushy na zewnątrz).
