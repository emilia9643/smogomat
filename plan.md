# Plan Projektu: Smogomat 🌬️📦

Projekt monitorowania jakości powietrza w oparciu o sieć czujników w automatach paczkowych (InPost Paczkomaty oraz Allegro One Box).
Projekt realizowany na zaliczenie przedmiotu **"Bazy danych"** oraz jako baza pod docelową aplikację webową/mobilną.

---

## 1. Założenia Architektoniczne i Stack Technologiczny

- **Baza danych:** PostgreSQL 16 + rozszerzenie przestrzenne **PostGIS 3.4**.
  - **Uruchomienie:** Kontener Docker Compose (`postgis/postgis:16-3.4`) ułatwiający uruchomienie na dowolnej maszynie i prezentację na uczelni.
- **Backend:** **Python 3.14** + **FastAPI** (architektura asynchroniczna).
- **Zarządzanie środowiskiem:** Standardowe wirtualne środowisko `.venv` + plik `requirements.txt`.
- **Podejście hybrydowe do bazy danych:**
  - **Dla backendu:** SQLAlchemy 2.0 (async) + GeoAlchemy2 + Alembic (deklaratywne modele ORM, walidacja Pydantic, migracje).
  - **Dla uczelni (zaliczenie):** Dedykowany katalog `sql/` z czystymi skryptami SQL:
    - Pełny DDL (tabele, relacje, klucze obce, więzy spójności).
    - Indeksy wydajnościowe B-Tree oraz indeksy przestrzenne **GiST** dla geometrii PostGIS.
    - Funkcje i triggery w języku **PL/pgSQL**.
    - Widoki standardowe oraz analityczne.

---

## 2. Model Danych i Struktura Bazy

### 2.1 Tabele

1. **`operators`**
   - `id` (PK, serial)
   - `code` (varchar, unique) – np. `INPOST`, `ALLEGRO`
   - `name` (varchar)
   - `website_url` (varchar)

2. **`voivodeships`**
   - `id` (PK, serial)
   - `name` (varchar, unique) – np. `dolnośląskie`

3. **`cities`**
   - `id` (PK, serial)
   - `voivodeship_id` (FK -> `voivodeships.id`)
   - `name` (varchar)
   - `slug` (varchar)

4. **`lockers`**
   - `id` (PK, serial)
   - `operator_id` (FK -> `operators.id`)
   - `city_id` (FK -> `cities.id`)
   - `locker_code` (varchar) – np. `WRO251M`, `AL086DWR` (UNIQUE w ramach operatora)
   - `external_id` (varchar, nullable) – wewnętrzne ID z API operatora
   - `street_address` (varchar)
   - `postal_code` (varchar)
   - `latitude` (numeric)
   - `longitude` (numeric)
   - `location_geom` (geometry(Point, 4326)) – punkt przestrzenny w standardzie WGS84
   - `description` (text, nullable)
   - `is_24_7` (boolean, default true)
   - `has_sensor` (boolean, default false)
   - `created_at` (timestamptz, default now())

5. **`locker_opening_hours`**
   - `id` (PK, serial)
   - `locker_id` (FK -> `lockers.id` ON DELETE CASCADE)
   - `day_of_week` (smallint) – 0-6 (poniedziałek-niedziela)
   - `open_time` (time)
   - `close_time` (time)

6. **`air_quality_levels`** (słownik znormalizowany wg skali GIOŚ)
   - `id` (PK, serial)
   - `code` (varchar, unique) – np. `VERY_GOOD`, `GOOD`, `MODERATE`, `SUFFICIENT`, `BAD`, `VERY_BAD`
   - `label_pl` (varchar) – *Bardzo dobry, Dobry, Umiarkowany, Dostateczny, Zły, Bardzo zły*
   - `color_hex` (varchar) – kody kolorów pod mapę (np. `#00E400`, `#FFFF00`, `#FF7E00`, `#FF0000`, `#99004C`, `#7E0023`)

7. **`sensor_types`**
   - `id` (PK, serial)
   - `code` (varchar, unique) – `PM10`, `PM25`, `PM1`, `TEMPERATURE`, `HUMIDITY`, `PRESSURE`
   - `name` (varchar)
   - `unit` (varchar) – `µg/m³`, `°C`, `%`, `hPa`
   - `norm_limit_daily` (numeric, nullable) – dopuszczalna norma dobowa (np. 50 dla PM10, 25 dla PM2.5)

8. **`locker_sensors`**
   - `id` (PK, serial)
   - `locker_id` (FK -> `lockers.id` ON DELETE CASCADE)
   - `sensor_type_id` (FK -> `sensor_types.id`)
   - `is_active` (boolean, default true)
   - `installed_at` (timestamptz, default now())
   - *Constraint:* UNIQUE (`locker_id`, `sensor_type_id`)

9. **`measurement_snapshots`** (rejestr pojedynczego odczytu z automatu)
   - `id` (PK, bigserial)
   - `locker_id` (FK -> `lockers.id` ON DELETE CASCADE)
   - `overall_quality_id` (FK -> `air_quality_levels.id`, nullable)
   - `measured_at` (timestamptz)
   - `fetched_at` (timestamptz, default now())

10. **`measurement_values`** (konkretne wartości parametrów w ramach danego snapshotu)
    - `id` (PK, bigserial)
    - `snapshot_id` (FK -> `measurement_snapshots.id` ON DELETE CASCADE)
    - `locker_sensor_id` (FK -> `locker_sensors.id`)
    - `value` (numeric)
    - `norm_percentage` (numeric, nullable) – % normy
    - `quality_level_id` (FK -> `air_quality_levels.id`, nullable)

11. **`users`** *(w schemacie pod wymogi bazy i kolejną fazę)*
    - `id` (PK, serial)
    - `username` (varchar, unique)
    - `email` (varchar, unique)
    - `password_hash` (varchar)
    - `role` (varchar, default 'user')
    - `created_at` (timestamptz, default now())

12. **`user_favorite_lockers`** *(w schemacie pod wymogi bazy i kolejną fazę)*
    - `id` (PK, serial)
    - `user_id` (FK -> `users.id` ON DELETE CASCADE)
    - `locker_id` (FK -> `lockers.id` ON DELETE CASCADE)
    - `alias_name` (varchar, nullable)
    - `created_at` (timestamptz, default now())
    - *Constraint:* UNIQUE (`user_id`, `locker_id`)

---

### 2.2 Specyfika Akademicka Bazy Danych (PL/pgSQL, Indeksy, Widoki)

1. **Indeksy:**
   - Przestrzenny: `CREATE INDEX idx_lockers_location_geom ON lockers USING GIST (location_geom);`
   - Czasowy dla odczytów: `CREATE INDEX idx_snapshots_locker_measured ON measurement_snapshots (locker_id, measured_at DESC);`
   - Klucze obce i unikalne kody.
2. **Triggery i procedury:**
   - **`trg_sync_locker_geom`**: Automatycznie uzupełnia/aktualizuje pole `location_geom = ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)` przy `INSERT` lub `UPDATE` współrzędnych.
   - **`trg_update_has_sensor`**: Aktualizuje flagę `has_sensor = true` w tabeli `lockers` po zarejestrowaniu aktywnego czujnika w `locker_sensors`.
3. **Widoki (Views):**
   - **`v_latest_measurements`**: Zwraca najświeższy pomiar jakości powietrza dla każdego automatu paczkowego (omijając konieczność wolnych podzapytań w backendzie).
   - **`v_city_air_stats`**: Agregacja średniego stężenia pyłów i wskaźników jakości powietrza z ostatnich 24h w podziale na miasta i województwa.

---

## 3. Integracja z API Zewnętrznymi i Zasilanie Danych

### 3.1 Źródła Danych

- **InPost:**
  - *Wszystkie punkty:* `https://inpost.pl/sites/default/files/points-pl.json?v=...` (zawiera m.in. kod `n`, współrzędne `l.a`, `l.o`, adres, godziny).
  - *Odczyt czujnika:* `https://inpost.pl/shipx-point-data/{external_id}/{code}/air_index_level`
  - *Format odczytów:* ogólny `air_index_level` oraz lista stringów `air_sensors` (`PM1:0.88:`, `PM25:1.38:9.22`, itp.).
- **Allegro One Box:**
  - *Punkty i klastry:* `https://edge.allegro.pl/pickup-points/map/rectangle?withAirQuality=true&filter=allegro-one&countryCode=PL...`
  - *Odczyt:* Obiekt `airQuality` zawarty bezpośrednio w punktach (lub `null`, gdy brak czujnika).

### 3.2 Pipeline Zasilania (Ingestion)

1. **Seed automatów (skrypt CLI):**
   - Domyślnie zoptymalizowany szybki seed dla wybranego obszaru (np. Wrocław) do pracy deweloperskiej.
   - Flaga `--all` do pełnego importu danych z całego kraju.
2. **Cykliczne pobieranie pomiarów:**
   - Scheduler/worker w tle odpytujący wyłącznie automaty z flagą `has_sensor = true` (np. co 30–60 minut), co eliminuje ryzyko blokady IP / limitów zapytań.

---

## 4. Zakres Backend API (FastAPI - Faza 1)

Wszystkie endpointy w pierwszej fazie skupione są na udostępnianiu danych przestrzennych i pomiarowych:

- `GET /api/v1/lockers/nearby` – wyszukiwanie automatów w zadanym promieniu $R$ (metry/km) od współrzędnych (`lat`, `lng`), z opcjonalnym filtrem `only_with_sensors=true`. Wykorzystuje PostGIS `ST_DWithin` i `ST_Distance`.
- `GET /api/v1/lockers/bbox` – automaty w zadanym prostokącie współrzędnych (dla widoku mapy).
- `GET /api/v1/lockers/geojson` – kolekcja **GeoJSON FeatureCollection** gotowa do bezpośredniego wyświetlenia w bibliotekach mapowych (Leaflet, Mapbox).
- `GET /api/v1/lockers/{id}` – pełne szczegóły wybranego automatu wraz z najnowszym pomiarem jakości powietrza.
- `GET /api/v1/lockers/{id}/history` – historia pomiarów automatu (ostatnie 24 godziny / 7 dni) do wykresów.
- `GET /api/v1/stats/cities` – rankingi i średnie zanieczyszczenie powietrza per miasto.
- `POST /api/v1/sync/measurements` – wyzwolenie natychmiastowego pobrania świeżych odczytów (np. dla potrzeb testów).

---

## 5. Struktura Katalogów Projektu

```text
smogomat/
├── .env.example
├── docker-compose.yml          # Kontener PostgreSQL + PostGIS 16
├── requirements.txt            # Zależności Python (FastAPI, SQLAlchemy, GeoAlchemy2, httpx, itp.)
├── plan.md                     # Niniejszy dokument ustaleń
├── stackDesc.md                # Notatka z linkami do API i opisem
├── sql/                        # Czyste skrypty SQL na uczelnię
│   ├── 01_schema.sql           # Tabele, relacje, typy
│   ├── 02_indexes.sql          # Indeksy B-Tree i GiST
│   ├── 03_triggers.sql         # Funkcje i triggery PL/pgSQL
│   └── 04_views.sql            # Widoki analityczne
├── alembic/                    # Migracje schematu SQLAlchemy
├── app/
│   ├── main.py                 # Inicjalizacja aplikacji FastAPI
│   ├── core/
│   │   ├── config.py           # Ustawienia z Pydantic Settings
│   │   └── database.py         # Asynchroniczny silnik i sesja SQLAlchemy
│   ├── models/                 # Modele ORM (SQLAlchemy + GeoAlchemy2)
│   ├── schemas/                # Schematy walidacji Pydantic (w tym GeoJSON)
│   ├── api/
│   │   └── v1/
│   │       ├── lockers.py      # Endpointy przestrzenne i punktów
│   │       ├── measurements.py # Endpointy odczytów i historii
│   │       └── stats.py        # Statystyki miast
│   ├── services/
│   │   ├── inpost_client.py    # Integracja z API InPost
│   │   ├── allegro_client.py   # Integracja z API Allegro
│   │   └── sync_service.py     # Logika przetwarzania i zapisu snapshotów
│   └── cli/
│       └── seed.py             # Narzędzie CLI do inicjalizacji danych
```

---

## 6. Kolejne Kroki Realizacji

1. **Przygotowanie konfiguracji:** Utworzenie `requirements.txt`, `docker-compose.yml` oraz `.env.example`.
2. **Warstwa bazodanowa:** Przygotowanie skryptów SQL (`sql/`) oraz modeli SQLAlchemy/GeoAlchemy2.
3. **Klienci API i Seedowanie:** Implementacja pobierania danych z InPost/Allegro i CLI seedującego.
4. **Endpointy FastAPI:** Budowa endpointów przestrzennych (PostGIS) i analitycznych.
5. **Scheduler:** Konfiguracja cyklicznego odpytywania aktywnych sensorów.
