# Konzept: Native iOS/iPadOS/macOS Investment-App

Inspiriert von Wealthfolio. Greenfield, kostenlos, App Store. Stack: SwiftUI ·
SwiftData · CloudKit · Swift Concurrency

---

## Rahmenbedingungen (festgelegte Entscheidungen)

- **Kein Import** aus Wealthfolio (keine CSV-/SQLite-Kompatibilität nötig)
- **App Store, kostenlos** (keine In-App-Käufe in v1)
- **Marktdaten v1:** nur Yahoo Finance (kein API-Key nötig)
- **Broker-Anbindung:** nicht in v1 (nur manuelle Eingabe)
- **v1-Scope:** Kern (Portfolio + Accounts + Holdings)
- **Mac:** SwiftUI native Mac target (keine Catalyst)

---

## Scope v1 (Kern)

- Accounts anlegen (SECURITIES, CASH, CRYPTO)
- Alle 14 Aktivitätstypen erfassen (BUY, SELL, DEPOSIT, WITHDRAWAL, DIVIDEND, …)
- Holdings-Berechnung aus Transaktionshistorie (Tracking-Mode: TRANSACTIONS)
- Portfolio-Dashboard: Gesamtwert, Allokation, heutiges P&L
- Kurse via Yahoo Finance (automatischer Sync, Hintergrund-Fetch)
- Multi-Currency + automatische FX-Kurse (Yahoo Finance)
- CloudKit Private Database Sync (alle Geräte des Users)
- Adaptive UI: iPhone (Stack) · iPad (Split) · Mac (native SwiftUI)

**Nicht in v1:** Goals, Income-Ansicht, Taxonomien, Broker-Connect,
AI-Assistent, Widgets

---

## Architektur

### Schichten

```
SwiftUI Views
    ↓ @Observable ViewModels
Service Layer  (PortfolioService, QuoteService, AccountService)
    ↓ async/await · Actors
Repository Layer  (SwiftData ModelContext)
    ↓
SwiftData (SQLite lokal) ←→ CloudKit Private DB
```

### Leitprinzipien (aus Wealthfolio übernommen)

- Local-first: alle Daten lokal in SwiftData, CloudKit ist Sync-Layer
- Monetäre Werte als `Decimal` (kein Float)
- Berechnete Holdings/Werte sind Caches — nie direkt editierbar
- Event-getriebene Neuberechnung nach jeder Datenänderung

---

## SwiftData Datenmodell

```swift
@Model class Account
  id: UUID
  name: String
  accountType: AccountType      // SECURITIES | CASH | CRYPTO
  currency: String              // ISO 4217
  isDefault: Bool
  isActive: Bool
  isArchived: Bool
  trackingMode: TrackingMode    // TRANSACTIONS | HOLDINGS
  group: String?
  activities: [Activity]        // @Relationship(deleteRule: .cascade)

@Model class Activity
  id: UUID
  account: Account              // @Relationship
  asset: Asset?                 // @Relationship (optional für Cash-Aktivitäten)
  activityType: ActivityType    // BUY|SELL|DEPOSIT|WITHDRAWAL|DIVIDEND|…
  activityTypeOverride: ActivityType?
  subtype: String?
  status: ActivityStatus        // POSTED|PENDING|DRAFT|VOID
  activityDate: Date
  quantity: Decimal?
  unitPrice: Decimal?
  amount: Decimal?
  fee: Decimal?
  currency: String
  fxRate: Decimal?
  notes: String?

@Model class Asset
  id: String                    // ticker symbol oder "FX:EUR/USD"
  kind: AssetKind               // EQUITY | CRYPTO | FX | METAL | CASH
  name: String?
  displayCode: String?
  quoteCurrency: String
  quoteMode: QuoteMode          // AUTO | MANUAL
  instrumentSymbol: String?
  instrumentExchange: String?   // MIC-Code
  quotes: [Quote]               // @Relationship(deleteRule: .cascade)

@Model class Quote
  id: String                    // "{assetId}_{YYYY-MM-DD}_{SOURCE}"
  asset: Asset                  // @Relationship
  day: Date
  open: Decimal?
  high: Decimal?
  low: Decimal?
  close: Decimal
  adjclose: Decimal?
  volume: Decimal?
  currency: String
  source: QuoteSource           // YAHOO | MANUAL | CALCULATED

@Model class AppSettings
  baseCurrency: String          // z.B. "EUR"
```

### CloudKit-Konfiguration

```swift
.modelContainer(
    for: [Account.self, Activity.self, Asset.self, Quote.self, AppSettings.self],
    cloudKitDatabase: .private("iCloud.com.yourapp.portfolio")
)
```

Conflict resolution: SwiftData last-write-wins.

---

## Service Layer

### PortfolioCalculationActor

```swift
actor PortfolioCalculationActor {
    func calculateHoldings(for account: Account) async -> [Holding]
    func calculatePortfolioValue(accounts: [Account], quotes: QuoteCache, baseCurrency: String) async -> PortfolioSnapshot
    func calculateCashBalance(for account: Account) -> [String: Decimal]
}
```

Kernlogik aus Wealthfolio portiert: FIFO-Kostenbasis, Activity-Compiler, TWR
(später)

### QuoteService

```swift
actor QuoteService {
    func syncQuotes(for assets: [Asset]) async throws
    func getLatestQuote(for asset: Asset) -> Quote?
    func getHistoricalQuotes(for asset: Asset, from: Date, to: Date) -> [Quote]
}
```

- Yahoo Finance via URLSession
- Sync-Kategorien: NEW / ACTIVE / NEEDS_BACKFILL / CLOSED (wie Wealthfolio)
- BGAppRefreshTask (iOS) / NSBackgroundActivityScheduler (Mac)

### FXService

```swift
actor FXService {
    func getRate(from: String, to: String, on: Date) -> Decimal?
    func convert(_ amount: Decimal, from: String, to: String, on: Date) -> Decimal?
}
```

- Graph+BFS-Logik (Swift-Port aus Wealthfolio)
- FX-Paare als Asset `kind = .fx`, Kurse via Yahoo ("EURUSD=X")
- Minor-Currency-Normalisierung: GBp→GBP (×0.01), ZAc→ZAR, ILA→ILS

---

## Navigation & UI-Struktur

### Adaptive Navigation

```
iPhone                  iPad / Mac
─────────────────       ──────────────────────────────
NavigationStack         NavigationSplitView
  Dashboard               Sidebar | Detail | Inspector
  Holdings                  Dashboard
  Activities                Holdings
  Accounts                  Activities
  Settings                  Accounts
                            Settings
```

### Haupt-Screens v1

| Screen         | Inhalt                                                       |
| -------------- | ------------------------------------------------------------ |
| Dashboard      | Gesamtwert-Karte, Tages-P&L, Allokations-Donut, Top-Holdings |
| Holdings       | Liste aller Positionen, Kurs/Wert/P&L pro Position           |
| Account Detail | Holdings, Cash-Balance, Aktivitätsliste                      |
| Activity Form  | Sheet: Aktivität erfassen (14 Typen, kontextsensitiv)        |
| Asset Search   | Symbol suchen via Yahoo Finance Search API                   |
| Settings       | Basiswährung, Account-Verwaltung                             |

### Design-Prinzipien (aus Wealthfolio)

- "Beautiful and Boring": klare Typographie, keine Ablenkungen
- Zahlen prominent, Farben sparsam (nur Gewinn/Verlust: grün/rot)
- Primäre Aktion immer erreichbar (Toolbar-Button für neue Aktivität)

---

## Kern-Berechnungslogik (Swift-Port aus Wealthfolio)

Referenz: `crates/core/src/portfolio/` und `crates/core/src/activities/`

### Activity Compiler

```
BUY       → CashPosting(-qty×price-fee) + HoldingPosting(+qty, costBasis)
SELL      → CashPosting(+qty×price-fee) + HoldingPosting(-qty, FIFO)
DEPOSIT   → CashPosting(+amount-fee), NetContribution(+amount)
DIVIDEND  → CashPosting(+amount-fee)
DRIP      → CashPosting(0) + HoldingPosting(+qty)
```

### Holdings Calculator

1. Alle POSTED Activities chronologisch sortieren
2. Activity Compiler auf jede Activity anwenden
3. Lot-Tracking per Asset (FIFO)
4. Cash-Balance pro Währung akkumulieren

---

## Entwicklungs-Roadmap

### Phase 1 — Datenfundament

- SwiftData Schema + CloudKit-Container
- Account CRUD (SwiftUI Forms)
- Activity CRUD (alle 14 Typen, kontextsensitives Formular)
- Activity Compiler + Holdings Calculator (Unit-Tests!)

### Phase 2 — Marktdaten

- Yahoo Finance API-Integration (URLSession, async/await)
- Quote Sync Service (Hintergrund-Tasks)
- FX Service + CurrencyConverter (BFS)

### Phase 3 — Portfolio UI

- Dashboard Screen
- Holdings Screen
- Account Detail Screen
- Portfolio-Bewertung mit echten Kursen

### Phase 4 — Mac & iPad polish

- NavigationSplitView optimieren
- Keyboard Shortcuts
- Toolbar-Anpassungen, Drag & Drop

### Phase 5 — v2 Features

- Goals, Income-Ansicht, Taxonomien, Widgets, Broker-Connect

---

## Offene Fragen

1. **Minimales iOS/macOS-Target?** — Empfehlung: iOS 18 / macOS 15
2. **App-Name & Bundle-ID?**
3. **Basiswährung beim Onboarding fixierbar oder später änderbar?**
4. **CSV-Import in v1?** — Nützlich für Onboarding, auch ohne
   Wealthfolio-Kompatibilität
5. **Yahoo Finance ToS?** — Rechtliche Grauzone bei kommerzieller Nutzung prüfen
6. **Familien-Sharing?** — CloudKit Shared Containers möglich, aber komplex
