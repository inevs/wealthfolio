# Wealthfolio – Onboarding-Notizen

Fragen und Antworten aus der ersten Session (2026-02-23).

---

## 1. Repository-Übersicht

### Architektur-Muster: Dual-Target über Adapter

`BUILD_TARGET=tauri` oder `BUILD_TARGET=web` wählt zur Build-Zeit die
Adapter-Implementierung:

```
apps/frontend/src/adapters/tauri/  → Rust via Tauri IPC
apps/frontend/src/adapters/web/    → REST HTTP-Calls an Axum
```

Vite löst `@/adapters` per `resolve.alias` auf — kein Runtime-Overhead.

### Rust-Crate-Schichten

```
apps/tauri + apps/server
    ↓
crates/core         ← Geschäftslogik
    ↓
crates/storage-sqlite ← Diesel ORM
crates/market-data    ← Externe Kursdaten-APIs
```

---

## 2. Datenmodell

Zentrale Tabellen:

| Tabelle                              | Zweck                                           |
| ------------------------------------ | ----------------------------------------------- |
| `accounts`                           | Depot-/Kontokonten                              |
| `activities`                         | Alle Transaktionen (Kauf, Verkauf, Dividende …) |
| `assets`                             | Wertpapiere / Währungspaare / Rohstoffe         |
| `quotes`                             | Historische & aktuelle Kurse (auch FX-Kurse)    |
| `holdings_snapshots`                 | Berechneter Cache: Bestände pro Tag             |
| `daily_account_valuation`            | Berechneter Cache: Kontowert pro Tag            |
| `goals` + `goals_allocation`         | Finanzielle Ziele                               |
| `contribution_limits`                | Beitragslimits (z.B. RRSP/TFSA)                 |
| `app_settings`                       | Key-Value-Store für Einstellungen               |
| `taxonomies` + `taxonomy_categories` | Klassifizierungssystem für Assets               |
| `ai_threads` + `ai_messages`         | KI-Assistent-Persistenz                         |
| `sync_outbox` + `sync_*`             | Device-Sync-System                              |

**Wichtig:** Monetäre Werte (`quantity`, `unit_price`, `amount` etc.) werden als
`Text` gespeichert → `rust_decimal` für Präzision.

---

## 3. Kursquellen

### Provider (Priorität absteigend)

| Provider        | Aktien | Crypto | Forex | Metalle |
| --------------- | ------ | ------ | ----- | ------- |
| Yahoo Finance   | Global | ja     | ja    | ja      |
| Alpha Vantage   | Global | ja     | ja    | nein    |
| MarketData.app  | nur US | nein   | nein  | nein    |
| Metal Price API | nein   | nein   | nein  | ja      |
| Finnhub         | US/EU  | nein   | nein  | nein    |

Yahoo Finance ist kostenlos, kein API-Key nötig. Andere Provider brauchen Keys
(in App-Einstellungen).

### Quote-ID-Format

`{asset_id}_{YYYY-MM-DD}_{SOURCE}` → deterministisch, kein Duplikat-Problem bei
Re-Sync.

### Sync-Kategorien

- `ACTIVE` (Prio 100): offene Position → täglich
- `NEW` (80): noch keine Kurse → ab erster Aktivität −45 Tage
- `NEEDS_BACKFILL` (70): historische Lücke
- `RECENTLY_CLOSED` (50): < 30 Tage nach Schließung
- `CLOSED` (0): kein Sync

---

## 4. Währungsbehandlung

### Minor-Currency-Normalisierung

| Code          | →     | Faktor |
| ------------- | ----- | ------ |
| `GBp` / `GBX` | `GBP` | 0.01   |
| `ZAc` / `ZAC` | `ZAR` | 0.01   |
| `ILA`         | `ILS` | 0.01   |

### CurrencyConverter

- Graph-basiert: BFS findet Umrechnungspfad auch über Zwischenwährungen (z.B.
  EUR→CAD über USD)
- Inverse Kurse werden automatisch berechnet und gespeichert
- "Nearest Neighbor": findet zeitlich nächsten verfügbaren Kurs für ein Datum

### FX-Kurse werden wie Wertpapier-Kurse behandelt

- Landen in derselben `quotes`-Tabelle
- Asset-ID-Format: `FX:EUR/USD`
- Werden automatisch registriert wenn Aktivität in Fremdwährung importiert wird

---

## 5. Sync-Frequenz

| Umgebung          | Trigger            | Frequenz                                |
| ----------------- | ------------------ | --------------------------------------- |
| Desktop (Tauri)   | Event-getrieben    | Bei App-Start + bei jeder Datenänderung |
| Web-Server (Axum) | Scheduler + Events | **alle 4 Stunden** + bei API-Änderungen |

Alle Schreiboperationen laufen durch einen `WriteActor` (serializiert
DB-Schreibzugriffe). Per-Asset-Locking verhindert parallele Sync-Läufe für
dasselbe Asset.

---

## 6. Account-Verwaltung

### Account-Typen

`SECURITIES`, `CASH`, `CRYPTOCURRENCY`

### Tracking-Modi

- **`TRANSACTIONS`**: Holdings aus Transaktionshistorie berechnet
- **`HOLDINGS`**: Holdings direkt importiert/manuell (z.B. für Altsysteme)

### Besonderheiten

- Währung ist nach Anlage **nicht mehr änderbar** (Repository schützt das Feld)
- Broker-verwaltete Felder (`provider`, `provider_account_id` etc.) können vom
  User-Form **nicht überschrieben** werden
- Löschen → CASCADE auf Aktivitäten → verwaiste Assets werden deaktiviert
- Jede Änderung → Outbox-Event für Device-Sync
