# Wealthfolio – Session Memory

## Projekt-Übersicht

Desktop + Web Investment Tracker. Local-first, kein Cloud-Zwang. Version
3.0.0-beta.5 (AGPL-3.0).

## Stack

- **Frontend:** React 19, TypeScript, Vite 7, React Router v7, TanStack Query
  v5, Zustand v5, Tailwind CSS v4, shadcn/ui
- **Backend:** Tauri v2 (Desktop), Axum (Web-Server), Rust, SQLite, Diesel ORM
  v2
- **Monorepo:** pnpm workspaces + Cargo workspace

## Wichtige Dateipfade

- Schema: `crates/storage-sqlite/src/schema.rs`
- Adapter-System: `apps/frontend/src/adapters/` (tauri/ vs web/)
- Kern-Domäne: `crates/core/src/`
- Frontend-Features: `apps/frontend/src/features/`
- Routen: `apps/frontend/src/routes.tsx`
- Architektur-Docs: `docs/architecture/`

## Dev-Kommandos

- Web-Modus: `pnpm run dev:web` (Vite :1420, Axum :8080)
- Desktop: `pnpm tauri dev`
- Tests: `pnpm test` | `cargo test`

## Passwort Web-Modus

`.env.web` enthält `WF_AUTH_PASSWORD_HASH`. Aktuell Placeholder-Hash (salt:
"somesalt"), Passwort vermutlich `password`. Zum Deaktivieren Zeile
auskommentieren.

## Details-Dokumente

- `onboarding.md` – Architektur, Datenmodell, Kurse, Währungen, Accounts
- `ios-app-concept.md` – Konzept native iOS/iPadOS/macOS App (SwiftUI,
  SwiftData, CloudKit)
