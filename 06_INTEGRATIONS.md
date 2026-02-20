# 06_INTEGRATIONS.md

## Cel
Opis techniczny integracji z systemami zewnętrznymi.

---

## 1. SATEL (ETHM-1 Plus Integration)

### 1.1 Protokół i Komunikacja
Integracja odbywa się bezpośrednio po protokole TCP/IP (port 10004) z modułem ETHM-1 Plus.
- **Protokół:** Oficjalny "Protokół integracji ETHM-1" (ramki binarne, start 0xFE 0xFE, CRC-16).
- **Tryb:** Klient TCP (Worker łączy się do modułu).
- **Ograniczenie:** Moduł pozwala na max 1 połączenie. Worker musi utrzymywać sesję (Keep-Alive) i wznawiać ją po zerwaniu.

### 1.2 Zakres Danych (MVP)
Worker odpytuje centralę (Polling) lub nasłuchuje zdarzeń:
1. **Stan Wejść (Zones):** Naruszenie, Sabotaż, Alarm, Awaria.
2. **Stan Stref (Partitions):** Uzbrojona, Rozbrojona, Alarm, Czas na wyjście.
3. **Stan Wyjść:** Załączone/Wyłączone (np. sterowanie bramą).
4. **Awarie Systemowe:** Brak zasilania 230V, Akumulator, Awaria linii tel.

### 1.3 Sterowanie (v2.0 / Faza 2)
Dzięki stałemu połączeniu TCP możliwy jest command flow:
1. Operator klika "Uzbrój" w App.
2. Backend wysyła komendę do RabbitMQ (`cmd_queue`).
3. Worker odbiera komendę, formatuje ramkę binarną (z kodem użytkownika).
4. Worker wysyła ramkę do ETHM-1.
5. Worker odbiera potwierdzenie (ACK/NACK) i aktualizuje status w Redis/App.

### 1.4 Zdarzenia Nieautoryzowanego Dostepu

Centrala SATEL generuje specjalne zdarzenia bezpieczenstwa, ktore Satel Worker musi odbierac i mapowac na typy systemowe:

| Zdarzenie SATEL | Event Code | Typ w systemie | Priorytet | Opis |
|---|---|---|---|---|
| 3 bledne hasla | Zone event (tamper context) | `UNAUTHORIZED_ACCESS` | WARNING/CRITICAL | 3 nieudane proby wprowadzenia kodu na manipulatorze/czytniku |
| Nieznany kod dostepu | Zone event (access context) | `UNAUTHORIZED_ACCESS` | WARNING/CRITICAL | Proba uzycia niezarejestrowanego kodu |

**Notatki:**
- Priorytet (WARNING vs CRITICAL) konfigurowalny per obiekt.
- **Two-Phase Priority:** Worker ustawia `default_priority` na podstawie kodu zdarzenia. Backend wzbogaca koncowe `priority` konsultujac `ZONES.priority_override`. Szczegoly: **13_EVENT_SCHEMAS.md, sekcja 2**.
- Zdarzenia te tworza Bundle Alarm z flaga `requires_note = true` (patrz **05_ALARM_LIFECYCLE.md, sekcja 6.1**).
- Dedup key: `{panel_id}:UNAUTHORIZED_ACCESS:{timestamp_minute}`

### 1.5 Handshake i Autoryzacja ETHM-1 (SAT-02)

Połączenie TCP z modułem ETHM-1 Plus wymaga sekwencji inicjalizacyjnej:

```
1. Worker → TCP connect do ETHM-1 (port 10004)
2. ETHM-1 ← odpowiada ramką identyfikacyjną (0xFE 0xFE + device info)
3. Worker → wysyła komendę autoryzacji (jeśli skonfigurowany hasłem integracji)
   → Ramka: 0xFE 0xFE + cmd 0x00 + integration_password (6 znaków)
4. ETHM-1 ← ACK (0xEF) lub NACK (0xED)
5. Worker → po ACK rozpoczyna polling (Full State Dump, patrz sekcja 1.2)
```

**Hasło integracji:**
- Ustawiane w DLOAD lub na manipulatorze centrali.
- Przechowywane jako Docker Secret: `satel_integration_password`.
- Per panel — każdy panel może mieć inne hasło.

**Timeout handshake:** 5 sekund. Brak odpowiedzi → reconnect z exponential backoff.

### 1.6 Kompatybilność Firmware ETHM-1 (SAT-03)

| Firmware ETHM-1 | Wspierane | Testowane | Uwagi |
|---|---|---|---|
| **v1.06+** | ✅ TAK | ✅ | Minimalna wersja — pełny protokół integracji |
| v1.04 - v1.05 | ⚠️ Częściowe | ❌ | Brak komendy odczytu nazw stref (0x04). Polling działa, ale nazwy stref = "Strefa {N}" |
| < v1.04 | ❌ NIE | ❌ | Niekompatybilny format ramek |

**Centrala INTEGRA:**

| Model | Wspierane | Testowane | Uwagi |
|---|---|---|---|
| INTEGRA 128 | ✅ TAK | ✅ | Pełen zakres: 128 wejść, 32 strefy |
| INTEGRA 64 | ✅ TAK | ❌ | 64 wejścia, reszta jak 128 |
| INTEGRA 32 | ✅ TAK | ❌ | 32 wejścia |
| INTEGRA 24 | ⚠️ Prawdopodobnie | ❌ | Wymaga weryfikacji |
| VERSA | ❌ NIE | ❌ | Inny protokół (ETHM-1 nie obsługuje) |

**Wymaganie:** Przy onboarding nowego obiektu, Technician MUSI zweryfikować wersję firmware ETHM-1 i odnotować ją w danych panelu.

### 1.7 DLOAD — Współdzielenie Połączenia (SAT-04)

**Problem:** ETHM-1 pozwala na max 1 połączenie TCP. Gdy serwisant chce użyć DLOAD (oprogramowanie konfiguracyjne SATEL), musi rozłączyć Worker.

**Rozwiązanie:** Endpoint API do kontrolowanego zwolnienia połączenia.

```
POST /api/satel/connection-release
{
  "panel_id": "PAT001",
  "reason": "DLOAD service session",
  "duration_minutes": 30,
  "requested_by": "user_id"
}

Response:
{
  "status": "released",
  "reconnect_at": "2026-02-16T15:00:00Z",
  "warning": "Worker nie będzie monitorował panelu PAT001 przez 30 min"
}
```

**Logika Workera:**
1. Otrzymuje sygnał "release" (przez Redis pub/sub).
2. Zamyka połączenie TCP z ETHM-1 dla danego panelu.
3. **NIE próbuje reconnect** przez `duration_minutes`.
4. Po upływie czasu — automatyczny reconnect + Full State Dump.
5. Event: `PANEL_CONNECTION_RELEASED` → `AUDIT_LOG`.

**Ograniczenia:**
- Max 60 min. Po 60 min Worker reconnect nawet bez jawnego sygnału.
- Tylko rola ADMIN/MASTER/TECHNICIAN może wywołać release.

### 1.8 Symulator ETHM-1 (SAT-01)

**Cel:** Umożliwić rozwój i testy Satel Worker bez fizycznego hardware.

| Element | Wartość |
|---|---|
| Typ | TCP server (Python, mock) |
| Port | 10004 (identyczny jak ETHM-1) |
| Protokół | Pełna emulacja ramek binarnych ETHM-1 |
| Scenariusze | Normalny polling, alarm (zone violation), tamper, auto-arm response, disconnect |
| CI/CD | Symulator startuje jako kontener w pipeline testowym |
| Tryby | `NORMAL` (odpowiada poprawnie), `FLAKY` (traci pakiety 10%), `TIMEOUT` (brak odpowiedzi co 5 ramkę) |

**Minimalny zakres emulacji:**
- Komendy 0x00-0x06 (odczyt stref, wejść, wyjść, awarii)
- ACK/NACK na komendy sterowania (0x80-0x84)
- Heartbeat (0x7F) — odpowiedź w < 500ms
- Scenariusz: wejście 1 w violation → Worker generuje event → test weryfikuje event w RabbitMQ

---

## 2. SMS (Efento / Bluelog)

### 2.1 Hardware
- **Modem GSM (USB/Serial):** Fizycznie podłączony do serwera.
- **Narzędzie:** `gammu-smsd` lub dedykowany skrypt Python (`pyserial`).
- Numer SIM dedykowany wyłącznie do odbioru alertów.

### 2.2 Zaufani Nadawcy i Bezpieczeństwo
System reaguje **TYLKO** na SMSy z 2 znanych numerów:
- **Numer 1:** Efento Cloud (alarmy temperatury lodówek/zamrażarek).
- **Numer 2:** Bluelog (alarmy temperatury chłodni/spedycji).

SMSy z innych numerów → **ignorowane** + zapis w logu systemowym.

- Mapowanie numerów przechowywane w tabeli `SMS_PHONE_MAPPING`.
- Niezidentyfikowane numery → log WARNING, ignore event.
- Raw SMS body **nie logowany** w produkcji (potencjalnie zawiera dane osobowe), jedynie hash MD5 treści.

### 2.3 Format SMS (Efento Cloud)

**Alarm (początek):**
```
2026-02-10 11:03:00 Alarm! Regula Leg_szczep_prawa_MIN, czujnik: Leg_szczep_prawa w Swiat Zdrowia Operat - Leg_Szczep, wartosc 1.7C
```

**Parsowanie:**
| Pole | Wartość z przykładu |
|---|---|
| Timestamp | `2026-02-10 11:03:00` |
| Typ | `Alarm!` → CRITICAL |
| Reguła | `Leg_szczep_prawa_MIN` |
| Czujnik | `Leg_szczep_prawa` → mapuj na `TEMP_SENSORS.name` |
| Lokalizacja | `Swiat Zdrowia Operat - Leg_Szczep` → mapuj na `OBJECTS.name` |
| Wartość | `1.7C` |

**Powrót do normy:**
```
2026-02-10 11:24:00 Powrot do normalnego stanu. Regula Leg_szczep_prawa_MIN, czujnik Leg_szczep_prawa w Swiat Zdrowia Operat - Leg_Szczep: Wartosc 2.0C
```
- Typ: `Powrot do normalnego stanu` → INFO (zamyka otwarty alarm temp.).

### 2.3a Jakość parsowania SMS (SIL-05)

> **Problem:** SMS może dotrzeć ucięty (160 znaków), zgarblowany (kodowanie) lub niekompletny. Parser powinien oznaczyć jakość parsowania aby operator wiedział czy może ufać danym.

**Pole `sms_quality` w event payload:**

| Wartość | Opis | Akcja |
|---|---|---|
| `complete` | SMS w pełni sparsowany, wszystkie pola wypełnione | Normalny flow |
| `truncated` | SMS ucięty (> 160 znaków brak terminatora), ale kluczowe dane obecne | Alarm tworzony, ale z flagą "dane mogą być niekompletne" |
| `garbled` | Kodowanie uszkodzone, parser wyciągnął co mógł | Alarm tworzony z priorytetem WARNING, wymaga ręcznej weryfikacji |
| `unparseable` | Parser nie rozpoznał formatu | Brak alarmu, log WARNING, raw SMS zapisany do AUDIT_LOG |

**Implementacja:** Parser ustawia `sms_quality` w event payload → Worker propaguje do `EVENTS.details` → UI wyświetla ikonę jakości.

Przykład oryginalnego SMS:
```
(Alertt) Gad Spedycja (S1, 21040DD5): Leg_szczep_prawa, -4.0°C
```

| Pole | Wartość |
|---|---|
| Lokalizacja | `Gad Spedycja` |
| Czujnik | `S1` / `21040DD5` |
| Pomiar | `Leg_szczep_prawa` |
| Temp | `-4.0°C` |
| Typ | `Alert temp.` → CRITICAL |

**Koniec alertu:**
```
Koniec alertu temp. dla rejestratora Gad Spedycja (S1, 21040DD5)
```
- Typ: `Koniec alertu` → INFO (zamyka otwarty alarm temp.).

### 2.5 Przepływ SMS → RabbitMQ
1. Modem odbiera SMS.
2. Skrypt sprawdza nadawcę → czy w `SMS_PHONE_MAPPING`?
3. Jeśli TAK → parser (Efento lub Bluelog) wyciąga dane + ustawia `sms_quality`.
4. Mapowanie czujnika → Obiekt (przez `TEMP_SENSORS`).
5. **PII Isolation:** SMS Agent hashuje pole `raw_sms` (SHA-256) → `raw_sms_hash`. Oryginalny tekst SMS zapisywany do tabeli `sms_raw_archive` (dostęp: MASTER/SYSTEM). Pole `raw_sms` **NIGDY** nie trafia do RabbitMQ ani `EVENTS.details`.
6. Tworzenie Raw Event (z polem `sms_quality` i `raw_sms_hash`) → RabbitMQ.
7. Worker tworzy / aktualizuje Bundle Alarm z flagą `requires_note = true`.

### 2.6 Fallback (Future)
- Bramka SMS online (API) w przypadku awarii modemu.

---

## 3. Mapy (Google Maps / OpenStreetMap)

### 3.1 Wyświetlanie
- Widget mapy we Flutterze.
- Markery (Pinezki) dla każdego Obiektu.
- Kolor markera zależny od statusu (Zielony = OK, Czerwony = Alarm).

### 3.2 Geokodowanie
- Przy dodawaniu obiektu, adres zamieniany jest na Lat/Lon.
- Możliwość ręcznej korekty pozycji pinezki (Drag & Drop).

---

## 4. CCTV (Tylko Metadane)

**NON-GOAL:** Nie robimy streamingu wideo w aplikacji.

### Zakres integracji:
- Przechowywanie danych w bazie:
  - Model rejestratora
  - Adres IP (LAN/WAN)
  - Login / Hasło (zaszyfrowane, Timed Reveal)
  - Ilość kanałów
- Przycisk "Kopiuj hasło" / "Otwórz w przeglądarce" (uruchamia zewnętrzną przeglądarkę).
