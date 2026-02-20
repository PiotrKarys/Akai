# 14_SATEL_COMMANDS

## Cel
Referencyjna tabela komend protokołu ETHM-1 używanych przez Satel Worker. Dokument oparty o oficjalną specyfikację "Protokół integracji ETHM-1".

> **Uwaga:** Poniższe kody komend odpowiadają oficjalnemu protokołowi. Szczegóły bitowe (format ramki, CRC) opisano w sekcji 4.

---

## 1. Komendy MVP (must-have do pierwszego połączenia)

### 1.1 Komendy odczytu stanu (polling)

| Cmd | Hex | Nazwa | Opis | Częstotliwość |
|---|---|---|---|---|
| New Data | 0x7F | Sprawdź zmiany | Zwraca flagę, czy coś się zmieniło od ostatniego zapytania. Podstawowa komenda heartbeat/polling. | Co 1s |
| Zones Violation | 0x00 | Stan wejść (naruszenie) | Bitfields — które wejścia są naruszone | Co 2s lub po "New Data" |
| Zones Tamper | 0x01 | Stan wejść (sabotaż) | Bitfields — które wejścia mają sabotaż | Co 5s |
| Zones Alarm | 0x02 | Stan wejść (alarm) | Bitfields — które wejścia są w alarmie | Co 2s |
| Zones Tamper Alarm | 0x03 | Alarm sabotażu wejść | Bitfields | Co 5s |
| Partitions State (byte 1) | 0x09 | Stan stref | Armed / Disarmed / Alarm / Entry delay / Exit delay | Co 2s |
| Partitions Alarm | 0x13 | Alarmy stref | Które strefy są w alarmie | Co 2s |
| Outputs State | 0x17 | Stan wyjść | ON / OFF per wyjście | Co 5s |
| Troubles Part 1-5 | 0x1C-0x20 | Awarie systemowe | AC loss, Battery, Tamper obudowy, etc. | Co 10s |

### 1.2 Inicjalizacja połączenia

Sekwencja startowa po nawiązaniu połączenia TCP:

1. **Connect** do ETHM-1 na port 10004 (TCP)
2. **Handshake/Auth** — jeśli moduł wymaga kodu integracji, wysłać komendę autoryzacji
3. **Full State Dump** — odpytać kolejno:
   - 0x00 (Zones Violation)
   - 0x01 (Zones Tamper)
   - 0x02 (Zones Alarm)
   - 0x09 (Partitions State)
   - 0x17 (Outputs State)
   - 0x1C-0x20 (Troubles)
4. **Zapisać w Redis** — pełny stan centrali
5. **Przejść w tryb polling** — 0x7F co 1s, szczegółowe komendy wg potrzeb

---

## 2. Komendy sterujące (v2.0)

| Cmd | Hex | Nazwa | Opis | Parametry |
|---|---|---|---|---|
| Arm Mode 0 | 0x80 | Uzbrojenie pełne | Full arm wybranych stref | user_code + partition_mask |
| Arm Mode 1 | 0x81 | Uzbrojenie nocne | Stay arm | user_code + partition_mask |
| Arm Mode 2 | 0x82 | Uzbrojenie bez wewnętrznych | Away arm (bez czujek wewnętrznych) | user_code + partition_mask |
| Disarm | 0x84 | Rozbrojenie | Disarm wybranych stref | user_code + partition_mask |
| Output On | 0x88 | Włącz wyjście | Załącz wyjście (np. bramę) | output_mask |
| Output Off | 0x89 | Wyłącz wyjście | Wyłącz wyjście | output_mask |
| Clear Alarm | 0x85 | Kasowanie alarmu | Reset alarmu w centrali | user_code + partition_mask |

---

## 3. Format Ramki

### Request (Worker → ETHM-1)

```
┌────────┬────────┬───────────┬──────────────┬──────────┬────────┐
│ 0xFE   │ 0xFE   │ CMD (1B)  │ DATA (0-nB)  │ CRC (2B) │ 0xFE   │
│ start  │ start  │           │              │ CRC-16   │ 0x0D   │
└────────┴────────┴───────────┴──────────────┴──────────┴────────┘
```

### Response (ETHM-1 → Worker)

```
┌────────┬────────┬───────────┬──────────────┬──────────┬────────┐
│ 0xFE   │ 0xFE   │ CMD (1B)  │ DATA (0-nB)  │ CRC (2B) │ 0xFE   │
│ start  │ start  │ echo/resp │              │ CRC-16   │ 0x0D   │
└────────┴────────┴───────────┴──────────────┴──────────┴────────┘
```

### Obliczanie CRC-16
- Algorytm: CRC-CCITT (zgodnie z dokumentacją Satel)
- Polynomial: 0x1021
- Init: 0x147A (specyficzne dla Satel)
- Obliczane z: CMD + DATA

### Bajt-stuffing
- Jeśli w danych występuje bajt 0xFE, musi być zdublowany (0xFE 0xFE) aby nie został zinterpretowany jako marker startu/stopu
- Worker musi dekodować stuffing przy odbieraniu i kodować przy wysyłaniu

---

## 4. Mapowanie Event Codes → Typy Alarmów

| Event Code (hex) | Typ w systemie | Priorytet | Opis |
|---|---|---|---|
| Zone violation (0x00 bit=1) | `ALARM` | CRITICAL | Naruszenie wejścia — włamanie |
| Zone tamper (0x01 bit=1) | `TAMPER` | CRITICAL | Sabotaż czujki |
| Zone alarm (0x02 bit=1) | `ALARM` | CRITICAL | Alarm aktywny na wejściu |
| Partition armed (0x09) | `ARM` | INFO | Strefa uzbrojona |
| Partition disarmed (0x09) | `DISARM` | INFO | Strefa rozbrojona |
| Partition alarm (0x13 bit=1) | `ALARM` | CRITICAL | Alarm na strefie |
| AC loss (0x1C bit) | `TROUBLE` | WARNING | Brak zasilania 230V |
| Battery low (0x1C bit) | `TROUBLE` | WARNING | Słaby akumulator |
| Comm trouble (0x1D bit) | `TROUBLE` | WARNING | Awaria komunikacji |
| 3 wrong passwords (keypad event) | `UNAUTHORIZED_ACCESS` | WARNING/CRITICAL | 3 bledne hasla na manipulatorze/czytniku |
| Unknown access code (keypad event) | `UNAUTHORIZED_ACCESS` | WARNING/CRITICAL | Proba uzycia niezarejestrowanego kodu |

---

## 5. Keep-Alive i Reconnect

### Heartbeat
- Worker wysyła 0x7F (New Data) co **1 sekundę**
- Jeśli brak odpowiedzi w ciągu **3 sekund** → uznaj połączenie za zerwane

### Reconnect Logic (Exponential Backoff)

| Próba | Delay | Akcja |
|---|---|---|
| 1 | 1s | Reconnect |
| 2 | 2s | Reconnect |
| 3 | 4s | Reconnect + alert WARNING |
| 4 | 8s | Reconnect |
| 5 | 16s | Reconnect + alert CRITICAL |
| 6+ | 30s | Reconnect (cap at 30s) |
| Po 5 min | — | Alert: "Panel {id} — brak komunikacji > 5 min" |

### Po reconnect
1. Full State Dump (sekcja 1.2)
2. Porównanie nowego stanu z ostatnim znanym (Redis)
3. Wygenerowanie delta events (co się zmieniło w trakcie downtime)
4. Publikacja delta events do RabbitMQ

---

## 6. Ograniczenia ETHM-1

| Ograniczenie | Wartość | Implikacja |
|---|---|---|
| Max połączeń TCP | **1** | Worker = exclusive, żadne inne narzędzie nie może się łączyć w tym samym czasie |
| Max stref | 32 (INTEGRA 128) | Bitfield: 4 bajty per komenda stref |
| Max wejść | 128 (INTEGRA 128) | Bitfield: 16 bajtów per komenda wejść |
| Max wyjść | 128 | Bitfield: 16 bajtów |
| Timeout odpowiedzi | ~2s | Jeśli brak odpowiedzi > 2s → retry raz, potem reconnect |

---

## 7. Fuzz Testing — ETHM-1 Parser (SAT-05)

> **Cel:** Upewnić się, że parser ramek binarnych nie crashuje, nie wchodzi w nieskończoną pętlę i nie leakuje pamięci przy złośliwych/uszkodzonych danych.

### 7.1 Zakres testów

| Scenariusz | Opis | Oczekiwany wynik |
|---|---|---|
| **Ramka bez terminatora** | 0xFE 0xFE + dane, brak kompletnej ramki | Timeout → reconnect, brak crash |
| **CRC nieprawidłowy** | Poprawna ramka z błędnym CRC-16 | Odrzucenie ramki, log WARNING, retry |
| **Ramka zerowa** | 100 bajtów 0x00 | Odrzucenie, nie wchodzi w parser |
| **Ramka gigantyczna** | 64KB danych po 0xFE 0xFE | Buffer limit reached → drop, log ERROR |
| **Ramka z ujemnymi wartościami** | Bitfield ze wszystkimi bitami = 1 | Parser traktuje jako "wszystkie strefy w alarmie" (poprawnie) |
| **Losowe bajty** | Randomowe 1-1000 bajtów | Brak crash, brak memory leak |
| **Replay attack** | Powtórzenie tej samej ramki 1000x w 1 sekundzie | Deduplikacja, brak flood do RabbitMQ |
| **Partial frame + disconnect** | Pół ramki → TCP disconnect | Cleanup buffera, reconnect |

### 7.2 Narzędzia

- **Python:** `hypothesis` library (property-based testing) z custom strategy dla ramek SATEL.
- **CI/CD:** Fuzz testy uruchamiane w pipeline z symulatororem ETHM-1 (tryb `FLAKY` + `TIMEOUT`).
- **Czas:** Min. 10 min fuzz per build (configurable).

### 7.3 Implementacja

```python
from hypothesis import given, strategies as st

# Generuj losowe "ramki" ETHM-1
satel_frame = st.binary(min_size=0, max_size=65535)

@given(raw_data=satel_frame)
def test_parser_never_crashes(raw_data: bytes):
    """Parser MUSI zwrócić ParsedFrame lub ParsingError — nigdy wyjątek."""
    result = parse_ethm1_frame(raw_data)
    assert isinstance(result, (ParsedFrame, ParsingError))
```

---

## 8. Connection Release API (Endpoint DLOAD)

> Pełna specyfikacja endpointu: `06_INTEGRATIONS.md`, sekcja 1.7.

Satel Worker respektuje sygnał `connection_release` dla danego panelu. Komendy sterowania wysłane w trakcie release zwracają `409 PANEL_RELEASED`:

```json
{
  "error": {
    "code": "PANEL_RELEASED",
    "message": "Panel PAT001 jest tymczasowo rozłączony (DLOAD session). Reconnect za 25 min.",
    "details": { "reconnect_at": "2026-02-16T15:00:00Z" }
  }
}
```
