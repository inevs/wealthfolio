# Wealthfolio: End-to-End Code Walkthrough

*2026-02-28T13:44:05Z by Showboat 0.6.1*
<!-- showboat-id: 6a6d5435-7fef-424a-bcc9-e3d0510a6ed6 -->

This document traces a single end-to-end flow through Wealthfolio's full stack: from app boot, through a user creating an account, all the way to the portfolio recalculation completing and the UI refreshing. The stack is React (Vite) + Tauri (Rust) + SQLite (Diesel), with a parallel Axum HTTP server for web-mode deployments. The same frontend code runs on both — the platform difference is erased at build time.

## 1. App Startup

Everything begins in `apps/tauri/src/lib.rs`. The `run()` function builds the Tauri application, registers plugins, and hands off to a platform-specific setup hook. On desktop, that hook calls `desktop_setup()`.

```bash
sed -n '55,120p' /Users/sven/development/wealthfolio/apps/tauri/src/lib.rs
```

```output

    /// Initializes desktop-specific plugins.
    pub fn init_plugins(handle: &AppHandle) {
        let _ = handle.plugin(tauri_plugin_updater::Builder::new().build());
        let _ = handle.plugin(tauri_plugin_window_state::Builder::new().build());
    }

    /// Performs synchronous setup on desktop: initializes context, menu, and registers listeners.
    pub fn setup(handle: AppHandle, app_data_dir: &str) -> Result<(), Box<dyn std::error::Error>> {
        // Initialize context synchronously (required before any commands can work)
        let init_result = tauri::async_runtime::block_on(async {
            context::initialize_context(app_data_dir).await
        })?;
        let context = Arc::new(init_result.context);
        let event_receiver = init_result.event_receiver;

        // Make context available to all commands
        handle.manage(Arc::clone(&context));

        // Start the domain event queue worker now that context is managed
        // This must be done in an async context since it spawns a tokio task
        let worker_handle = handle.clone();
        let worker_context = Arc::clone(&context);
        tauri::async_runtime::spawn(async move {
            domain_events::TauriDomainEventSink::start_queue_worker(
                event_receiver,
                worker_handle,
                worker_context,
            );
        });

        // Menu setup is synchronous (no I/O)
        setup_menu(&handle, &context.instance_id);

        // Notify frontend that app is ready
        // The frontend will trigger the initial portfolio update and update check after it's mounted
        emit_app_ready(&handle);

        // Trigger startup sync (async, non-blocking)
        // After this, user manually triggers sync via button
        let startup_handle = handle.clone();
        let startup_context = Arc::clone(&context);
        tauri::async_runtime::spawn(async move {
            scheduler::run_startup_sync(&startup_handle, &startup_context).await;
        });

        // Start background device sync engine (self-skips when device is not READY).
        #[cfg(feature = "device-sync")]
        {
            let device_sync_context = Arc::clone(&context);
            tauri::async_runtime::spawn(async move {
                if let Err(err) = crate::commands::device_sync::ensure_background_engine_started(
                    device_sync_context,
                )
                .await
                {
                    log::warn!("Failed to start background device sync engine: {}", err);
                }
            });
        }

        Ok(())
    }
}

// ─────────────────────────────────────────────────────────────────────────────
```

Five things happen in sequence, synchronously: (1) `context::initialize_context()` builds the entire service graph and returns it; (2) the context is handed to Tauri's state manager so every command can access it; (3) the domain event queue worker is spawned as a background tokio task; (4) a startup menu and ready signal go to the frontend; (5) background jobs (broker sync, device sync) are spawned and run independently. The critical dependency: the context must exist before any IPC command can run.

## 2. Building the Service Graph

`context::initialize_context()` is a long function in `apps/tauri/src/context/providers.rs`. It wires up the entire dependency graph — database pool, repositories, and services — and returns a `ServiceContext` plus the domain event channel receiver.

```bash
sed -n '1,100p' /Users/sven/development/wealthfolio/apps/tauri/src/context/providers.rs
```

```output
use super::ai_environment::TauriAiEnvironment;
use super::registry::ServiceContext;
use crate::domain_events::TauriDomainEventSink;
use crate::secret_store::shared_secret_store;
use crate::services::ConnectService;
use std::sync::{Arc, RwLock};
use tokio::sync::mpsc;
use wealthfolio_ai::{AiProviderService, ChatConfig, ChatService};
use wealthfolio_connect::{
    BrokerSyncService, CoreImportRunRepositoryAdapter, ImportRunRepositoryTrait,
};
use wealthfolio_core::{
    accounts::AccountService,
    activities::ActivityService,
    assets::{AlternativeAssetService, AssetClassificationService, AssetService},
    events::DomainEvent,
    fx::{FxService, FxServiceTrait},
    goals::GoalService,
    health::HealthService,
    limits::ContributionLimitService,
    portfolio::{
        allocation::AllocationService,
        holdings::{HoldingsService, HoldingsValuationService},
        income::IncomeService,
        net_worth::NetWorthService,
        performance::PerformanceService,
        snapshot::SnapshotService,
        valuation::ValuationService,
    },
    quotes::{QuoteService, QuoteServiceTrait},
    settings::{SettingsRepositoryTrait, SettingsService, SettingsServiceTrait},
    taxonomies::TaxonomyService,
};
use wealthfolio_device_sync::{engine::DeviceSyncRuntimeState, DeviceEnrollService};
use wealthfolio_storage_sqlite::{
    accounts::AccountRepository,
    activities::ActivityRepository,
    ai_chat::AiChatRepository,
    assets::{AlternativeAssetRepository, AssetRepository},
    db::{self, write_actor},
    fx::FxRepository,
    goals::GoalRepository,
    health::HealthDismissalRepository,
    limits::ContributionLimitRepository,
    market_data::{MarketDataRepository, QuoteSyncStateRepository},
    portfolio::{snapshot::SnapshotRepository, valuation::ValuationRepository},
    settings::SettingsRepository,
    sync::{AppSyncRepository, BrokerSyncStateRepository, ImportRunRepository, PlatformRepository},
    taxonomies::TaxonomyRepository,
};

/// Result of context initialization, including the receiver for domain events.
pub struct ContextInitResult {
    pub context: ServiceContext,
    pub event_receiver: mpsc::UnboundedReceiver<DomainEvent>,
}

pub async fn initialize_context(
    app_data_dir: &str,
) -> Result<ContextInitResult, Box<dyn std::error::Error>> {
    let db_path = db::init(app_data_dir)?;
    db::run_migrations(&db_path)?;

    let pool = db::create_pool(&db_path)?;
    let writer = write_actor::spawn_writer(pool.as_ref().clone());

    // Instantiate Repositories
    let settings_repository = Arc::new(SettingsRepository::new(pool.clone(), writer.clone()));
    let account_repository = Arc::new(AccountRepository::new(pool.clone(), writer.clone()));
    let activity_repository = Arc::new(ActivityRepository::new(pool.clone(), writer.clone()));
    let asset_repository = Arc::new(AssetRepository::new(pool.clone(), writer.clone()));
    let goal_repo = Arc::new(GoalRepository::new(pool.clone(), writer.clone()));
    let market_data_repo = Arc::new(MarketDataRepository::new(pool.clone(), writer.clone()));
    let limit_repository = Arc::new(ContributionLimitRepository::new(
        pool.clone(),
        writer.clone(),
    ));
    let fx_repository = Arc::new(FxRepository::new(pool.clone(), writer.clone()));
    let snapshot_repository = Arc::new(SnapshotRepository::new(pool.clone(), writer.clone()));
    let app_sync_repository = Arc::new(AppSyncRepository::new(pool.clone(), writer.clone()));
    let valuation_repository = Arc::new(ValuationRepository::new(pool.clone(), writer.clone()));
    let platform_repository = Arc::new(PlatformRepository::new(pool.clone(), writer.clone()));
    let broker_sync_state_repository =
        Arc::new(BrokerSyncStateRepository::new(pool.clone(), writer.clone()));

    // Domain event sink - TauriDomainEventSink sends events to a channel
    // The worker will be started by the caller after the context is managed
    // Must be created before services that emit events
    let (domain_event_sink, event_receiver) = TauriDomainEventSink::new();
    let domain_event_sink: Arc<dyn wealthfolio_core::events::DomainEventSink> =
        Arc::new(domain_event_sink);

    let fx_service =
        Arc::new(FxService::new(fx_repository.clone()).with_event_sink(domain_event_sink.clone()));
    fx_service.initialize()?;

    let settings_service = Arc::new(SettingsService::new(
        settings_repository.clone(),
        fx_service.clone(),
    ));
```

The pattern repeats for every service: create a repository (concrete SQLite type), then create a service (wraps the repository behind a trait object). Two architectural choices stand out here:

**Write actor**: `write_actor::spawn_writer(pool)` returns a `WriteHandle` — a channel to a single dedicated tokio task. All SQLite writes go through this actor. This sidesteps SQLite's single-writer limitation without using a mutex.

**Domain event channel**: `TauriDomainEventSink::new()` returns `(sink, receiver)`. The sink is injected into every service that needs to announce mutations. The receiver goes to the queue worker. Services never touch the worker; they just fire-and-forget into the channel.

```bash
grep -n 'pub struct ServiceContext' /Users/sven/development/wealthfolio/apps/tauri/src/context/registry.rs
```

```output
18:pub struct ServiceContext {
```

```bash
sed -n '18,60p' /Users/sven/development/wealthfolio/apps/tauri/src/context/registry.rs
```

```output
pub struct ServiceContext {
    pub base_currency: Arc<RwLock<String>>,
    pub instance_id: Arc<String>,

    /// Domain event sink for emitting events after mutations.
    /// Runtime bridges (Tauri/Web) implement this to trigger portfolio recalculation,
    /// asset enrichment, and broker sync based on domain events.
    /// Note: The sink is used by services injected at construction time; this field
    /// is kept for documentation and possible future access patterns.
    #[allow(dead_code)]
    pub domain_event_sink: Arc<dyn DomainEventSink>,

    // Services
    pub settings_service: Arc<dyn settings::SettingsServiceTrait>,
    pub activity_service: Arc<dyn activities::ActivityServiceTrait>,
    pub account_service: Arc<dyn accounts::AccountServiceTrait>,
    pub goal_service: Arc<dyn goals::GoalServiceTrait>,
    pub asset_service: Arc<dyn assets::AssetServiceTrait>,
    pub quote_service: Arc<dyn quotes::QuoteServiceTrait>,
    pub limits_service: Arc<dyn limits::ContributionLimitServiceTrait>,
    pub fx_service: Arc<dyn fx::FxServiceTrait>,
    pub performance_service: Arc<dyn portfolio::performance::PerformanceServiceTrait>,
    pub income_service: Arc<dyn portfolio::income::IncomeServiceTrait>,
    pub snapshot_service: Arc<dyn portfolio::snapshot::SnapshotServiceTrait>,
    pub snapshot_repository: Arc<SnapshotRepository>,
    pub app_sync_repository: Arc<AppSyncRepository>,
    pub holdings_service: Arc<dyn portfolio::holdings::HoldingsServiceTrait>,
    pub allocation_service: Arc<dyn portfolio::allocation::AllocationServiceTrait>,
    pub valuation_service: Arc<dyn portfolio::valuation::ValuationServiceTrait>,
    pub net_worth_service: Arc<dyn portfolio::net_worth::NetWorthServiceTrait>,
    pub sync_service: Arc<dyn BrokerSyncServiceTrait>,
    pub alternative_asset_service: Arc<dyn AlternativeAssetServiceTrait>,
    pub taxonomy_service: Arc<dyn taxonomies::TaxonomyServiceTrait>,
    pub connect_service: Arc<ConnectService>,
    pub ai_provider_service: Arc<dyn AiProviderServiceTrait>,
    pub ai_chat_service: Arc<ChatService<TauriAiEnvironment>>,
    pub device_enroll_service: Arc<DeviceEnrollService>,
    pub device_sync_runtime: Arc<DeviceSyncRuntimeState>,
    pub health_service: Arc<health::HealthService>,
}

impl ServiceContext {
    pub fn get_base_currency(&self) -> String {
```

Every field is `Arc<dyn SomeTrait>` — a reference-counted trait object. Commands get a clone of `Arc<ServiceContext>` and call `state.account_service()` to access the service they need. The `Arc` means no copying; the trait object means Tauri commands depend on abstractions, not concrete SQLite types. Swapping implementations (e.g. for testing) only requires changing `providers.rs`.

## 3. IPC Command Registration

Back in `lib.rs`, Tauri's `invoke_handler!` macro registers all the commands the frontend can call. There are over 100 of them, spread across ~20 modules.

```bash
grep -n 'invoke_handler\|generate_handler' /Users/sven/development/wealthfolio/apps/tauri/src/lib.rs | head -5
```

```output
247:        .invoke_handler(tauri::generate_handler![
```

```bash
sed -n '247,280p' /Users/sven/development/wealthfolio/apps/tauri/src/lib.rs
```

```output
        .invoke_handler(tauri::generate_handler![
            // Account commands
            commands::account::get_accounts,
            commands::account::create_account,
            commands::account::update_account,
            commands::account::delete_account,
            // Activity commands
            commands::activity::search_activities,
            commands::activity::create_activity,
            commands::activity::update_activity,
            commands::activity::save_activities,
            commands::activity::delete_activity,
            commands::activity::check_activities_import,
            commands::activity::import_activities,
            commands::activity::get_account_import_mapping,
            commands::activity::save_account_import_mapping,
            commands::activity::check_existing_duplicates,
            commands::activity::parse_csv,
            // Settings commands
            commands::settings::get_settings,
            commands::settings::is_auto_update_check_enabled,
            commands::settings::update_settings,
            commands::settings::get_latest_exchange_rates,
            commands::settings::update_exchange_rate,
            commands::settings::add_exchange_rate,
            commands::settings::delete_exchange_rate,
            // Goal commands
            commands::goal::create_goal,
            commands::goal::update_goal,
            commands::goal::delete_goal,
            commands::goal::get_goals,
            commands::goal::update_goal_allocations,
            commands::goal::load_goals_allocations,
            // Portfolio commands
```

The macro generates a match arm for every string name like `"get_accounts"`. When the frontend calls `invoke("get_accounts", ...)`, Tauri deserializes the JSON payload into the command's parameters and dispatches to the right function. The command name on the wire is the function name snake_cased.

Here's what a command looks like. This is `apps/tauri/src/commands/account.rs`:

```bash
sed -n '1,75p' /Users/sven/development/wealthfolio/apps/tauri/src/commands/account.rs
```

```output
use std::sync::Arc;

use crate::context::ServiceContext;
use log::{debug, error};
use tauri::State;

use wealthfolio_core::accounts::{Account, AccountUpdate, NewAccount};

#[tauri::command]
pub async fn get_accounts(
    include_archived: Option<bool>,
    state: State<'_, Arc<ServiceContext>>,
) -> Result<Vec<Account>, String> {
    debug!("Fetching accounts...");
    let include = include_archived.unwrap_or(false);
    if include {
        state
            .account_service()
            .get_all_accounts()
            .map_err(|e| format!("Failed to load accounts: {}", e))
    } else {
        state
            .account_service()
            .get_non_archived_accounts()
            .map_err(|e| format!("Failed to load accounts: {}", e))
    }
}

#[tauri::command]
pub async fn create_account(
    account: NewAccount,
    state: State<'_, Arc<ServiceContext>>,
) -> Result<Account, String> {
    debug!("Adding new account...");
    // Domain events handle recalculation automatically
    state
        .account_service()
        .create_account(account)
        .await
        .map_err(|e| {
            error!("Failed to add new account: {}", e);
            format!("Failed to add new account: {}", e)
        })
}

#[tauri::command]
pub async fn update_account(
    account_update: AccountUpdate,
    state: State<'_, Arc<ServiceContext>>,
) -> Result<Account, String> {
    debug!("Updating account {:?}...", account_update.id);

    // Domain events handle recalculation automatically
    state
        .account_service()
        .update_account(account_update.clone())
        .await
        .map_err(|e| format!("Failed to update account {:?}: {}", account_update.id, e))
}

#[tauri::command]
pub async fn delete_account(
    account_id: String,
    state: State<'_, Arc<ServiceContext>>,
) -> Result<(), String> {
    debug!("Deleting account {}...", account_id);
    // Domain events handle recalculation automatically
    state
        .account_service()
        .delete_account(&account_id)
        .await
        .map_err(|e| {
            error!("Failed to delete account {}: {}", account_id, e);
            e.to_string()
        })
```

Commands are intentionally thin. Every command has the same shape: accept parameters + `State<'_, Arc<ServiceContext>>`, call one service method, map the error to a `String`. The comment "Domain events handle recalculation automatically" appears throughout — commands don't trigger portfolio updates directly; the service does that by emitting an event.

## 4. The Frontend: Build-Time Platform Erasure

The frontend is in `apps/frontend/`. It targets two runtimes — Tauri IPC and plain HTTP — but ships one codebase. The trick is in `vite.config.ts`:

```bash
grep -n 'BUILD_TARGET\|#platform\|adapters\|alias' /Users/sven/development/wealthfolio/apps/frontend/vite.config.ts | head -20
```

```output
28:// Default to "tauri" for local development - use BUILD_TARGET=web for web builds
30:const buildTarget = process.env.BUILD_TARGET || "tauri";
41:    __BUILD_TARGET__: JSON.stringify(buildTarget),
44:    alias: {
47:      // Conditional adapter alias based on build target
48:      "@/adapters": path.resolve(
50:        buildTarget === "tauri" ? "./src/adapters/tauri" : "./src/adapters/web",
52:      // Platform-specific core module for shared adapters
53:      "#platform": path.resolve(
55:        buildTarget === "tauri" ? "./src/adapters/tauri/core" : "./src/adapters/web/core",
```

```bash
sed -n '28,60p' /Users/sven/development/wealthfolio/apps/frontend/vite.config.ts
```

```output
// Default to "tauri" for local development - use BUILD_TARGET=web for web builds
// TAURI_DEV_HOST is only set for mobile/network dev, so we can't rely on it
const buildTarget = process.env.BUILD_TARGET || "tauri";

// https://vitejs.dev/config/
export default defineConfig({
  envDir: "../..",
  plugins: [react(), tailwindcss()],
  publicDir: "public",
  optimizeDeps: {
    include: ["lucide-react", "recharts", "lodash"],
  },
  define: {
    __BUILD_TARGET__: JSON.stringify(buildTarget),
  },
  resolve: {
    alias: {
      "@wealthfolio/addon-sdk": path.resolve(__dirname, "../../packages/addon-sdk/src"),
      "@wealthfolio/ui": path.resolve(__dirname, "../../packages/ui/src"),
      // Conditional adapter alias based on build target
      "@/adapters": path.resolve(
        __dirname,
        buildTarget === "tauri" ? "./src/adapters/tauri" : "./src/adapters/web",
      ),
      // Platform-specific core module for shared adapters
      "#platform": path.resolve(
        __dirname,
        buildTarget === "tauri" ? "./src/adapters/tauri/core" : "./src/adapters/web/core",
      ),
      "@": path.resolve(__dirname, "./src"),
    },
    extensions: [".js", ".ts", ".jsx", ".tsx", ".json"],
  },
```

Two aliases do all the work:

- `@/adapters` resolves to either `adapters/tauri/` or `adapters/web/` — used by page components to import service functions.
- `#platform` resolves to either `adapters/tauri/core.ts` or `adapters/web/core.ts` — used inside shared adapters to get `invoke`, `logger`, and `isDesktop`.

Pages import from `@/adapters`. Shared adapters import from `#platform`. At build time both aliases collapse to a concrete path, so the final bundle contains zero conditional logic around platform differences.

Here's the Tauri `core.ts` — what `#platform` resolves to in a desktop build:

```bash
cat /Users/sven/development/wealthfolio/apps/frontend/src/adapters/tauri/core.ts
```

```output
// Core utilities for Tauri adapter - internal use only
import { invoke as tauriInvoke } from "@tauri-apps/api/core";
import { debug, error, info, trace, warn } from "@tauri-apps/plugin-log";

import type { Logger } from "../types";

/**
 * Logger implementation using Tauri's log plugin
 * Wraps the Tauri log functions to match the Logger interface
 */
export const logger: Logger = {
  error: (...args: unknown[]) => {
    error(args.map(String).join(" "));
  },
  warn: (...args: unknown[]) => {
    warn(args.map(String).join(" "));
  },
  info: (...args: unknown[]) => {
    info(args.map(String).join(" "));
  },
  debug: (...args: unknown[]) => {
    debug(args.map(String).join(" "));
  },
  trace: (...args: unknown[]) => {
    trace(args.map(String).join(" "));
  },
};

/**
 * Invoke a Tauri command (internal - use typed adapter functions instead)
 */
export const invoke = async <T>(command: string, payload?: Record<string, unknown>): Promise<T> => {
  try {
    return await tauriInvoke<T>(command, payload);
  } catch (err) {
    logger.error(`[Invoke] Command "${command}" failed: ${err}`);
    throw err;
  }
};

// Re-export tauriInvoke for cases where we need direct access (e.g., Channel streaming)
export { tauriInvoke };

// Platform detection flags for shared modules
export const isDesktop = true;
export const isWeb = false;
```

And the web equivalent — what `#platform` resolves to for a web/server build. The interface is identical; the implementation sends HTTP instead.

```bash
sed -n '1,80p' /Users/sven/development/wealthfolio/apps/frontend/src/adapters/web/core.ts
```

```output
// Web adapter core - Internal invoke function, COMMANDS map, and helpers
// This module exports invoke, logger, and platform constants for shared modules

import { getAuthToken, notifyUnauthorized } from "@/lib/auth-token";
import type { Logger } from "../types";

/** True when running in the desktop (Tauri) environment */
export const isDesktop = false;

/** True when running in the web environment */
export const isWeb = true;

export const API_PREFIX = "/api/v1";
export const EVENTS_ENDPOINT = `${API_PREFIX}/events/stream`;
export const AI_CHAT_STREAM_ENDPOINT = `${API_PREFIX}/ai/chat/stream`;

type CommandMap = Record<string, { method: string; path: string }>;

export const COMMANDS: CommandMap = {
  get_accounts: { method: "GET", path: "/accounts" },
  create_account: { method: "POST", path: "/accounts" },
  update_account: { method: "PUT", path: "/accounts" },
  delete_account: { method: "DELETE", path: "/accounts" },
  get_settings: { method: "GET", path: "/settings" },
  update_settings: { method: "PUT", path: "/settings" },
  is_auto_update_check_enabled: { method: "GET", path: "/settings/auto-update-enabled" },
  get_app_info: { method: "GET", path: "/app/info" },
  check_update: { method: "GET", path: "/app/check-update" },
  backup_database: { method: "POST", path: "/utilities/database/backup" },
  backup_database_to_path: { method: "POST", path: "/utilities/database/backup-to-path" },
  restore_database: { method: "POST", path: "/utilities/database/restore" },
  get_holdings: { method: "GET", path: "/holdings" },
  get_holding: { method: "GET", path: "/holdings/item" },
  get_asset_holdings: { method: "GET", path: "/holdings/by-asset" },
  get_historical_valuations: { method: "GET", path: "/valuations/history" },
  get_latest_valuations: { method: "GET", path: "/valuations/latest" },
  get_portfolio_allocations: { method: "GET", path: "/allocations" },
  // Snapshot management
  get_snapshots: { method: "GET", path: "/snapshots" },
  get_snapshot_by_date: { method: "GET", path: "/snapshots/holdings" },
  delete_snapshot: { method: "DELETE", path: "/snapshots" },
  save_manual_holdings: { method: "POST", path: "/snapshots" },
  import_holdings_csv: { method: "POST", path: "/snapshots/import" },
  check_holdings_import: { method: "POST", path: "/snapshots/import/check" },
  update_portfolio: { method: "POST", path: "/portfolio/update" },
  recalculate_portfolio: { method: "POST", path: "/portfolio/recalculate" },
  // Performance
  calculate_accounts_simple_performance: { method: "POST", path: "/performance/accounts/simple" },
  calculate_performance_history: { method: "POST", path: "/performance/history" },
  calculate_performance_summary: { method: "POST", path: "/performance/summary" },
  get_income_summary: { method: "GET", path: "/income/summary" },
  // Goals
  get_goals: { method: "GET", path: "/goals" },
  create_goal: { method: "POST", path: "/goals" },
  update_goal: { method: "PUT", path: "/goals" },
  delete_goal: { method: "DELETE", path: "/goals" },
  update_goal_allocations: { method: "POST", path: "/goals/allocations" },
  load_goals_allocations: { method: "GET", path: "/goals/allocations" },
  // FX
  get_latest_exchange_rates: { method: "GET", path: "/exchange-rates/latest" },
  update_exchange_rate: { method: "PUT", path: "/exchange-rates" },
  add_exchange_rate: { method: "POST", path: "/exchange-rates" },
  delete_exchange_rate: { method: "DELETE", path: "/exchange-rates" },
  // Activities
  search_activities: { method: "POST", path: "/activities/search" },
  create_activity: { method: "POST", path: "/activities" },
  update_activity: { method: "PUT", path: "/activities" },
  save_activities: { method: "POST", path: "/activities/bulk" },
  delete_activity: { method: "DELETE", path: "/activities" },
  // Activity import
  check_activities_import: { method: "POST", path: "/activities/import/check" },
  import_activities: { method: "POST", path: "/activities/import" },
  get_account_import_mapping: { method: "GET", path: "/activities/import/mapping" },
  save_account_import_mapping: { method: "POST", path: "/activities/import/mapping" },
  // Market data providers
  get_exchanges: { method: "GET", path: "/exchanges" },
  get_market_data_providers: { method: "GET", path: "/providers" },
  get_market_data_providers_settings: { method: "GET", path: "/providers/settings" },
  update_market_data_provider_settings: { method: "PUT", path: "/providers/settings" },
  // Contribution limits
```

```bash
grep -n 'export const invoke' /Users/sven/development/wealthfolio/apps/frontend/src/adapters/web/core.ts
```

```output
299:export const invoke = async <T>(command: string, payload?: Record<string, unknown>): Promise<T> => {
```

```bash
sed -n '299,360p' /Users/sven/development/wealthfolio/apps/frontend/src/adapters/web/core.ts
```

```output
export const invoke = async <T>(command: string, payload?: Record<string, unknown>): Promise<T> => {
  const config = COMMANDS[command];
  if (!config) throw new Error(`Unsupported command ${command}`);
  let url = `${API_PREFIX}${config.path}`;
  let body: BodyInit | undefined;

  switch (command) {
    case "update_account": {
      const data = payload as { accountUpdate: { id: string } & Record<string, unknown> };
      url += `/${data.accountUpdate.id}`;
      body = JSON.stringify(data.accountUpdate);
      break;
    }
    case "delete_account": {
      const data = payload as { accountId: string };
      url += `/${data.accountId}`;
      break;
    }
    case "create_account": {
      const data = payload as { account: Record<string, unknown> };
      body = JSON.stringify(data.account);
      break;
    }
    case "backup_database_to_path": {
      const { backupDir } = payload as { backupDir: string };
      body = JSON.stringify({ backupDir });
      break;
    }
    case "restore_database": {
      const { backupFilePath } = payload as { backupFilePath: string };
      body = JSON.stringify({ backupFilePath });
      break;
    }
    case "update_settings": {
      const data = payload as { settingsUpdate: Record<string, unknown> };
      body = JSON.stringify(data.settingsUpdate);
      break;
    }
    case "get_holdings": {
      const p = payload as { accountId: string };
      url += `?accountId=${encodeURIComponent(p.accountId)}`;
      break;
    }
    case "get_holding": {
      const { accountId, assetId } = payload as { accountId: string; assetId: string };
      const params = new URLSearchParams();
      params.set("accountId", accountId);
      params.set("assetId", assetId);
      url += `?${params.toString()}`;
      break;
    }
    case "get_asset_holdings": {
      const p = payload as { assetId: string };
      url += `?assetId=${encodeURIComponent(p.assetId)}`;
      break;
    }
    case "get_historical_valuations": {
      const p = payload as { accountId?: string; startDate?: string; endDate?: string };
      const params = new URLSearchParams();
      if (p?.accountId) params.set("accountId", p.accountId);
      if (p?.startDate) params.set("startDate", p.startDate);
      if (p?.endDate) params.set("endDate", p.endDate);
```

The web `invoke` is a big switch statement that maps the same command names used by Tauri to HTTP method + URL + body. The COMMANDS map declares the route; the switch handles argument marshalling (e.g. path params vs query params vs JSON body). From the shared adapter's perspective, calling `invoke("get_accounts", {})` looks identical regardless of platform.

## 5. A Request in Flight: Shared Adapters

Shared adapters live in `apps/frontend/src/adapters/shared/`. They're the single source of truth for how the frontend calls each operation, regardless of platform.

```bash
cat /Users/sven/development/wealthfolio/apps/frontend/src/adapters/shared/accounts.ts
```

```output
// Account Commands
import type { Account } from "@/lib/types";
import type { newAccountSchema } from "@/lib/schemas";
import type z from "zod";

import { invoke, logger, isDesktop } from "./platform";

type NewAccount = z.infer<typeof newAccountSchema>;

export const getAccounts = async (includeArchived?: boolean): Promise<Account[]> => {
  try {
    return await invoke<Account[]>("get_accounts", { includeArchived: includeArchived ?? false });
  } catch (error) {
    logger.error("Error fetching accounts.");
    throw error;
  }
};

export const createAccount = async (account: NewAccount): Promise<Account> => {
  try {
    return await invoke<Account>("create_account", { account });
  } catch (error) {
    logger.error("Error creating account.");
    throw error;
  }
};

export const updateAccount = async (account: NewAccount): Promise<Account> => {
  try {
    // Platform-aware: desktop strips currency (immutable after creation)
    const payload = isDesktop
      ? (() => {
          const { currency: _, ...rest } = account;
          return rest;
        })()
      : account;
    return await invoke<Account>("update_account", { accountUpdate: payload });
  } catch (error) {
    logger.error("Error updating account.");
    throw error;
  }
};

export const deleteAccount = async (accountId: string): Promise<void> => {
  try {
    await invoke<void>("delete_account", { accountId });
  } catch (error) {
    logger.error("Error deleting account.");
    throw error;
  }
};
```

Notice the import: `from "./platform"` — this is a local re-export that forwards from `#platform`, which Vite resolves to the right core. The adapter has one tiny bit of runtime branching (`isDesktop` check in `updateAccount`) to strip the `currency` field, which is immutable on Tauri but editable on web.

Page components import these adapter functions directly and pass them to React Query's `queryFn`. No Redux, no custom state machine — React Query handles caching, invalidation, and background refetching.

## 6. The Core Service Layer

The Tauri command calls `account_service().create_account(account)`. The service is in `crates/core/src/accounts/`. Here's the trait first:

```bash
grep -n 'trait AccountServiceTrait\|trait AccountRepositoryTrait' /Users/sven/development/wealthfolio/crates/core/src/accounts/accounts_traits.rs
```

```output
17:pub trait AccountRepositoryTrait: Send + Sync {
53:pub trait AccountServiceTrait: Send + Sync {
```

```bash
sed -n '17,90p' /Users/sven/development/wealthfolio/crates/core/src/accounts/accounts_traits.rs
```

```output
pub trait AccountRepositoryTrait: Send + Sync {
    /// Creates a new account.
    ///
    /// The implementation handles transaction management internally.
    async fn create(&self, new_account: NewAccount) -> Result<Account>;

    /// Updates an existing account.
    async fn update(&self, account_update: AccountUpdate) -> Result<Account>;

    /// Deletes an account by its ID.
    ///
    /// Returns the number of deleted records.
    async fn delete(&self, account_id: &str) -> Result<usize>;

    /// Retrieves an account by its ID.
    fn get_by_id(&self, account_id: &str) -> Result<Account>;

    /// Lists accounts with optional filters.
    ///
    /// # Arguments
    /// * `is_active_filter` - If Some, filter by active status
    /// * `is_archived_filter` - If Some, filter by archived status
    /// * `account_ids` - If Some, filter to only these account IDs
    fn list(
        &self,
        is_active_filter: Option<bool>,
        is_archived_filter: Option<bool>,
        account_ids: Option<&[String]>,
    ) -> Result<Vec<Account>>;
}

/// Trait defining the contract for Account service operations.
///
/// The service layer handles business logic and coordinates between
/// repositories and other services.
#[async_trait]
pub trait AccountServiceTrait: Send + Sync {
    /// Creates a new account with business validation.
    async fn create_account(&self, new_account: NewAccount) -> Result<Account>;

    /// Updates an existing account with business validation.
    async fn update_account(&self, account_update: AccountUpdate) -> Result<Account>;

    /// Deletes an account and handles related cleanup.
    async fn delete_account(&self, account_id: &str) -> Result<()>;

    /// Retrieves an account by ID.
    fn get_account(&self, account_id: &str) -> Result<Account>;

    /// Lists accounts with optional filters.
    fn list_accounts(
        &self,
        is_active_filter: Option<bool>,
        is_archived_filter: Option<bool>,
        account_ids: Option<&[String]>,
    ) -> Result<Vec<Account>>;

    /// Gets all accounts regardless of status.
    fn get_all_accounts(&self) -> Result<Vec<Account>>;

    /// Gets only active accounts.
    fn get_active_accounts(&self) -> Result<Vec<Account>>;

    /// Gets accounts by a list of IDs.
    fn get_accounts_by_ids(&self, account_ids: &[String]) -> Result<Vec<Account>>;

    /// Returns all non-archived accounts (for aggregates/history)
    fn get_non_archived_accounts(&self) -> Result<Vec<Account>>;

    /// Returns active, non-archived accounts (for UI selectors)
    fn get_active_non_archived_accounts(&self) -> Result<Vec<Account>>;

    /// Returns the configured base currency if available.
    fn get_base_currency(&self) -> Option<String>;
```

Two separate traits: `AccountRepositoryTrait` (data access) and `AccountServiceTrait` (business logic). The service holds an `Arc<dyn AccountRepositoryTrait>` — it never knows it's talking to SQLite. This layering means the core crate has zero knowledge of Diesel, SQLite, or Tauri.

```bash
sed -n '1,100p' /Users/sven/development/wealthfolio/crates/core/src/accounts/accounts_service.rs
```

```output
//! Account service implementation.

use log::{debug, info, warn};
use std::sync::{Arc, RwLock};

use super::accounts_model::{Account, AccountUpdate, NewAccount};
use super::accounts_traits::{AccountRepositoryTrait, AccountServiceTrait};
use crate::assets::AssetRepositoryTrait;
use crate::errors::Result;
use crate::events::{CurrencyChange, DomainEvent, DomainEventSink};
use crate::fx::FxServiceTrait;
use crate::quotes::sync_state::SyncStateStore;

/// Service for managing accounts.
pub struct AccountService {
    repository: Arc<dyn AccountRepositoryTrait>,
    fx_service: Arc<dyn FxServiceTrait>,
    base_currency: Arc<RwLock<String>>,
    event_sink: Arc<dyn DomainEventSink>,
    asset_repository: Arc<dyn AssetRepositoryTrait>,
    sync_state_store: Arc<dyn SyncStateStore>,
}

impl AccountService {
    /// Creates a new AccountService instance.
    pub fn new(
        repository: Arc<dyn AccountRepositoryTrait>,
        fx_service: Arc<dyn FxServiceTrait>,
        base_currency: Arc<RwLock<String>>,
        event_sink: Arc<dyn DomainEventSink>,
        asset_repository: Arc<dyn AssetRepositoryTrait>,
        sync_state_store: Arc<dyn SyncStateStore>,
    ) -> Self {
        Self {
            repository,
            fx_service,
            base_currency,
            event_sink,
            asset_repository,
            sync_state_store,
        }
    }
}

#[async_trait::async_trait]
impl AccountServiceTrait for AccountService {
    /// Creates a new account with currency exchange support.
    async fn create_account(&self, new_account: NewAccount) -> Result<Account> {
        let base_currency = self.base_currency.read().unwrap().clone();
        debug!(
            "Creating account..., base_currency: {}, new_account.currency: {}",
            base_currency, new_account.currency
        );

        // Perform async currency pair registration if needed
        if new_account.currency != base_currency {
            self.fx_service
                .register_currency_pair(new_account.currency.as_str(), base_currency.as_str())
                .await?;
        }

        // Repository handles transaction internally
        let result = self.repository.create(new_account).await?;

        // Emit AccountsChanged event with currency info for FX sync planning
        let currency_changes = vec![CurrencyChange {
            account_id: result.id.clone(),
            old_currency: None,
            new_currency: result.currency.clone(),
        }];
        self.event_sink.emit(DomainEvent::accounts_changed(
            vec![result.id.clone()],
            currency_changes,
        ));

        Ok(result)
    }

    /// Updates an existing account.
    async fn update_account(&self, account_update: AccountUpdate) -> Result<Account> {
        // Get existing account to detect changes
        let account_id = account_update.id.as_ref().ok_or_else(|| {
            crate::Error::Validation(crate::errors::ValidationError::InvalidInput(
                "Account ID is required".to_string(),
            ))
        })?;
        let existing = self.repository.get_by_id(account_id)?;

        let result = self.repository.update(account_update).await?;

        // Detect currency changes and register FX pair if needed
        let currency_changes = if existing.currency != result.currency {
            let base_currency = self.base_currency.read().unwrap().clone();
            if result.currency != base_currency {
                self.fx_service
                    .register_currency_pair(result.currency.as_str(), base_currency.as_str())
                    .await?;
            }
            vec![CurrencyChange {
                account_id: result.id.clone(),
```

The service does three things:
1. **Business logic** — validate currency, register an FX pair with the FX service if the account uses a non-base currency.
2. **Delegate persistence** — call `self.repository.create()`, which handles the SQL transaction internally.
3. **Announce the mutation** — call `self.event_sink.emit(DomainEvent::accounts_changed(...))`. This is fire-and-forget: `emit` returns immediately. The service doesn't wait for any recalculation.

## 7. Persistence: Diesel + the Write Actor

The repository is in `crates/storage-sqlite/src/accounts/`. It uses Diesel ORM and writes through a dedicated actor.

```bash
find /Users/sven/development/wealthfolio/crates/storage-sqlite/src/accounts -name '*.rs' | sort
```

```output
/Users/sven/development/wealthfolio/crates/storage-sqlite/src/accounts/mod.rs
/Users/sven/development/wealthfolio/crates/storage-sqlite/src/accounts/model.rs
/Users/sven/development/wealthfolio/crates/storage-sqlite/src/accounts/repository.rs
```

```bash
sed -n '1,80p' /Users/sven/development/wealthfolio/crates/storage-sqlite/src/accounts/repository.rs
```

```output
use async_trait::async_trait;
use diesel::prelude::*;
use diesel::r2d2::{self, Pool};
use diesel::sqlite::SqliteConnection;
use std::sync::Arc;

use crate::db::{get_connection, WriteHandle};
use crate::errors::StorageError;
use crate::schema::accounts;
use crate::schema::accounts::dsl::*;

use super::model::AccountDB;
use wealthfolio_core::accounts::{Account, AccountRepositoryTrait, AccountUpdate, NewAccount};
use wealthfolio_core::errors::Result;

/// Repository for managing account data in the database
pub struct AccountRepository {
    pool: Arc<Pool<r2d2::ConnectionManager<SqliteConnection>>>,
    writer: WriteHandle,
}

impl AccountRepository {
    /// Creates a new AccountRepository instance
    pub fn new(
        pool: Arc<Pool<r2d2::ConnectionManager<SqliteConnection>>>,
        writer: WriteHandle,
    ) -> Self {
        Self { pool, writer }
    }
}

// Implement the trait
#[async_trait]
impl AccountRepositoryTrait for AccountRepository {
    /// Creates a new account
    async fn create(&self, new_account: NewAccount) -> Result<Account> {
        new_account.validate()?;

        self.writer
            .exec_tx(move |tx| {
                let mut account_db: AccountDB = new_account.into();
                account_db.id = uuid::Uuid::new_v4().to_string();

                diesel::insert_into(accounts::table)
                    .values(&account_db)
                    .execute(tx.conn())
                    .map_err(StorageError::from)?;

                let payload_db = account_db.clone();
                let account: Account = account_db.into();
                tx.insert(&payload_db)?;

                Ok(account)
            })
            .await
    }

    async fn update(&self, account_update: AccountUpdate) -> Result<Account> {
        account_update.validate()?;

        // Capture which optional fields were explicitly set before conversion
        let is_archived_provided = account_update.is_archived.is_some();
        let tracking_mode_provided = account_update.tracking_mode.is_some();

        self.writer
            .exec_tx(move |tx| {
                let mut account_db: AccountDB = account_update.into();

                let existing = accounts
                    .select(AccountDB::as_select())
                    .find(&account_db.id)
                    .first::<AccountDB>(tx.conn())
                    .map_err(StorageError::from)?;

                // Preserve fields that shouldn't change
                account_db.currency = existing.currency;
                account_db.created_at = existing.created_at;
                account_db.updated_at = chrono::Utc::now().naive_utc();

                // Preserve broker-managed fields (only set by broker sync, not user form)
```

Writes use `self.writer.exec_tx(|tx| { ... })`. The `WriteHandle` is a channel to the write actor. `exec_tx` sends the closure over the channel, the actor runs it on its thread with a SQLite transaction, and sends back the result. This serializes all writes through one thread — safe for SQLite's single-writer model — while keeping the async API intact for callers.

The closure gets a `tx` with two operations: `tx.conn()` (the live Diesel connection) and `tx.insert(&payload_db)` (which logs to a sync event journal for the device-sync feature). Both run inside the same transaction.

```bash
grep -n 'pub fn spawn_writer\|pub struct WriteHandle\|pub fn exec_tx' /Users/sven/development/wealthfolio/crates/storage-sqlite/src/db/write_actor.rs | head -10
```

```output
18:pub struct WriteHandle {
201:pub fn spawn_writer(pool: DbPool) -> WriteHandle {
```

```bash
sed -n '18,50p' /Users/sven/development/wealthfolio/crates/storage-sqlite/src/db/write_actor.rs
```

```output
pub struct WriteHandle {
    // Sender part of the MPSC channel to send jobs.
    // Each job is a boxed closure, and a oneshot sender is used for the reply.
    // The Box<dyn Any + Send> is used for type erasure of the job's return type.
    #[allow(clippy::type_complexity)]
    tx: mpsc::Sender<(
        Job<Box<dyn Any + Send + 'static>>,
        oneshot::Sender<Result<Box<dyn Any + Send + 'static>>>,
    )>,
}

impl WriteHandle {
    /// Executes a database job on the writer actor's dedicated connection.
    ///
    /// # Arguments
    /// * `job`: A closure that takes a mutable reference to `SqliteConnection`
    ///   and performs database operations.
    ///
    /// # Returns
    /// A `Result<T>` containing the outcome of the job.
    pub async fn exec<F, T>(&self, job: F) -> Result<T>
    where
        F: FnOnce(&mut SqliteConnection) -> Result<T> + Send + 'static,
        T: Send + 'static + Any, // Add Any bound for T
    {
        // Create a oneshot channel for receiving the result from the actor.
        let (ret_tx, ret_rx) = oneshot::channel();

        // Send the job to the writer actor.
        // The job is wrapped to return a Box<dyn Any + Send> for type erasure.
        self.tx
            .send((
                Box::new(move |c| job(c).map(|v| Box::new(v) as Box<dyn Any + Send>)), // Cast to Box<dyn Any + Send>
```

The write actor uses a classic pattern: a channel of `(Job, oneshot::Sender)` pairs. The handle sends a closure; the actor runs it; the result comes back through the oneshot channel. Type erasure (`Box<dyn Any + Send>`) lets the channel carry any return type without generics on the channel itself. The actor thread holds the single SQLite write connection for the process lifetime.

## 8. Domain Events: The Async Reaction System

After the repository write completes, control returns to the service, which calls `self.event_sink.emit(DomainEvent::accounts_changed(...))`. Let's look at what `emit` actually does.

```bash
cat /Users/sven/development/wealthfolio/apps/tauri/src/domain_events/sink.rs
```

```output
//! Tauri domain event sink implementation.
//!
//! Implements DomainEventSink by sending events to an mpsc channel
//! for async processing by the queue worker.

use std::sync::Arc;

use log::{error, info};
use tauri::AppHandle;
use tokio::sync::mpsc;
use wealthfolio_core::events::{DomainEvent, DomainEventSink};

use super::queue_worker::event_queue_worker;
use crate::context::ServiceContext;

/// Tauri implementation of the domain event sink.
///
/// Events are sent to an unbounded mpsc channel for processing by a background
/// worker task. This ensures `emit()` is fast and non-blocking.
pub struct TauriDomainEventSink {
    sender: mpsc::UnboundedSender<DomainEvent>,
}

impl TauriDomainEventSink {
    /// Creates a new TauriDomainEventSink with only the sender.
    ///
    /// The queue worker must be started separately using `start_queue_worker`
    /// after the ServiceContext is fully initialized.
    pub fn new() -> (Self, mpsc::UnboundedReceiver<DomainEvent>) {
        let (sender, receiver) = mpsc::unbounded_channel();
        info!("TauriDomainEventSink created (worker must be started separately)");
        (Self { sender }, receiver)
    }

    /// Starts the queue worker with the receiver, app handle, and context.
    ///
    /// This should be called after ServiceContext is fully initialized and
    /// the AppHandle is available.
    pub fn start_queue_worker(
        receiver: mpsc::UnboundedReceiver<DomainEvent>,
        app_handle: AppHandle,
        context: Arc<ServiceContext>,
    ) {
        tokio::spawn(event_queue_worker(receiver, app_handle, context));
        info!("TauriDomainEventSink queue worker started");
    }
}

impl Default for TauriDomainEventSink {
    fn default() -> Self {
        let (sink, _receiver) = Self::new();
        // Note: receiver is dropped here, so events won't be processed
        // This is only for trait compatibility; real usage should use new()
        sink
    }
}

impl DomainEventSink for TauriDomainEventSink {
    fn emit(&self, event: DomainEvent) {
        if let Err(e) = self.sender.send(event) {
            error!("Failed to send domain event to queue: {}", e);
        }
    }

    fn emit_batch(&self, events: Vec<DomainEvent>) {
        for event in events {
            if let Err(e) = self.sender.send(event) {
                error!("Failed to send domain event to queue: {}", e);
                // Continue trying to send remaining events
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_new_returns_sender_and_receiver() {
        let (sink, mut receiver) = TauriDomainEventSink::new();

        let event = DomainEvent::AssetsCreated {
            asset_ids: vec!["AAPL".to_string()],
        };

        sink.emit(event);

        // Check that event was sent
        let received = receiver.try_recv();
        assert!(received.is_ok());

        match received.unwrap() {
            DomainEvent::AssetsCreated { asset_ids } => {
                assert_eq!(asset_ids, vec!["AAPL".to_string()]);
            }
            _ => panic!("Expected AssetsCreated event"),
        }
    }

    #[test]
    fn test_emit_batch_sends_all_events() {
        let (sink, mut receiver) = TauriDomainEventSink::new();

        let events = vec![
            DomainEvent::AssetsCreated {
                asset_ids: vec!["AAPL".to_string()],
            },
            DomainEvent::AssetsCreated {
                asset_ids: vec!["MSFT".to_string()],
            },
        ];

        sink.emit_batch(events);

        // Check that both events were sent
        assert!(receiver.try_recv().is_ok());
        assert!(receiver.try_recv().is_ok());
        assert!(receiver.try_recv().is_err()); // No more events
    }
}
```

`emit()` is a single `sender.send(event)` call into an unbounded mpsc channel. It's synchronous, non-blocking, and always fast. The service calling it doesn't know or care what happens next — that's handled entirely by the queue worker running in a separate tokio task. The sink and receiver are created together at startup but the worker is started separately, after the `ServiceContext` is fully initialized and managed by Tauri.

## 9. The Queue Worker: Debounce and Dispatch

The queue worker runs forever in a tokio task, consuming from the mpsc receiver.

```bash
sed -n '1,120p' /Users/sven/development/wealthfolio/apps/tauri/src/domain_events/queue_worker.rs
```

```output
//! Event queue worker that processes domain events with debouncing.
//!
//! Receives events via an mpsc channel, debounces them within a 500ms window,
//! and then processes the batch to trigger platform-specific actions.

use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::time::Duration;

use log::{debug, error, info, warn};
use tauri::{AppHandle, Emitter};
use tokio::sync::mpsc;
use wealthfolio_core::constants::PORTFOLIO_TOTAL_ACCOUNT_ID;
use wealthfolio_core::events::DomainEvent;
use wealthfolio_core::health::HealthServiceTrait;

#[cfg(feature = "connect-sync")]
use super::planner::plan_broker_sync;
use super::planner::{plan_asset_enrichment, plan_portfolio_job};
#[cfg(feature = "connect-sync")]
use crate::commands::brokers_sync::perform_broker_sync;
use crate::context::ServiceContext;
use crate::events::{
    MarketSyncResult, PortfolioRequestPayload, MARKET_SYNC_COMPLETE, MARKET_SYNC_ERROR,
    MARKET_SYNC_START, PORTFOLIO_UPDATE_COMPLETE, PORTFOLIO_UPDATE_ERROR, PORTFOLIO_UPDATE_START,
};

/// Debounce window duration in milliseconds.
const DEBOUNCE_MS: u64 = 1000;

/// Runs the event queue worker that processes domain events with debouncing.
///
/// This function:
/// 1. Receives events from the mpsc channel
/// 2. Collects events until a 500ms debounce window expires
/// 3. Processes the batch of events by calling planner functions
/// 4. Triggers appropriate actions (portfolio recalc, enrichment, broker sync)
///
/// Uses an `is_processing` guard to prevent new batches from being processed
/// while a previous batch (e.g., broker sync or portfolio recalc) is still running.
pub async fn event_queue_worker(
    mut receiver: mpsc::UnboundedReceiver<DomainEvent>,
    app_handle: AppHandle,
    context: Arc<ServiceContext>,
) {
    let debounce_duration = Duration::from_millis(DEBOUNCE_MS);
    let mut event_buffer: Vec<DomainEvent> = Vec::new();
    let is_processing = Arc::new(AtomicBool::new(false));

    loop {
        // If buffer is empty, wait indefinitely for the first event
        // If buffer has events, wait with a timeout for more events
        let maybe_event = if event_buffer.is_empty() {
            // Wait indefinitely for the first event
            receiver.recv().await
        } else {
            // Wait for more events or timeout
            tokio::select! {
                event = receiver.recv() => event,
                _ = tokio::time::sleep(debounce_duration) => None,
            }
        };

        match maybe_event {
            Some(event) => {
                // Add event to buffer and continue collecting
                event_buffer.push(event);
            }
            None if !event_buffer.is_empty() => {
                // Timeout expired or channel closed with events in buffer
                // Check if we're still processing a previous batch
                if is_processing.load(Ordering::SeqCst) {
                    // Still processing, keep collecting events
                    debug!("Debounce expired but previous batch still processing, continuing to collect events");
                    continue;
                }

                // Process the batch
                let events = std::mem::take(&mut event_buffer);
                is_processing.store(true, Ordering::SeqCst);
                process_event_batch(&events, &app_handle, &context).await;
                is_processing.store(false, Ordering::SeqCst);
            }
            None => {
                // Channel closed and buffer is empty - exit the worker
                // Wait for any in-progress processing to complete
                while is_processing.load(Ordering::SeqCst) {
                    tokio::time::sleep(Duration::from_millis(50)).await;
                }
                info!("Domain event queue worker shutting down");
                break;
            }
        }
    }
}

/// Processes a batch of domain events by planning and triggering actions.
async fn process_event_batch(
    events: &[DomainEvent],
    app_handle: &AppHandle,
    context: &Arc<ServiceContext>,
) {
    if events.is_empty() {
        return;
    }

    info!("Processing batch of {} domain events", events.len());

    // Plan and run portfolio job directly (not via event emission)
    // This ensures the is_processing guard properly tracks completion
    if let Some(payload) = plan_portfolio_job(events) {
        info!(
            "Running portfolio job (accounts: {:?})",
            payload.account_ids
        );
        run_portfolio_job(app_handle, context, payload).await;
    }

    // Plan and run asset enrichment (spawned as background task)
    let enrichment_asset_ids = plan_asset_enrichment(events);
```

The debounce logic: the worker waits indefinitely for the first event. Once it has one, it switches to a 1-second timeout loop, collecting more events. When the timeout fires (no event arrived in 1s), it processes the entire buffer as a batch.

The `is_processing` flag prevents re-entrant batches — if a portfolio recalc is still running when the debounce fires again, the worker keeps collecting into the buffer instead of kicking off a second job. Events from rapid mutations (creating 10 activities in a loop) all land in one batch and trigger one recalculation.

```bash
cat /Users/sven/development/wealthfolio/apps/tauri/src/domain_events/planner.rs
```

```output
//! Event planning functions for translating domain events into actions.
//!
//! These functions analyze batches of domain events and determine what
//! platform-specific actions need to be triggered.

use std::collections::HashSet;

use wealthfolio_core::accounts::TrackingMode;
use wealthfolio_core::events::DomainEvent;

use crate::events::PortfolioRequestPayload;

/// Plans a portfolio recalculation job from a batch of domain events.
///
/// Merges account_ids and asset_ids from:
/// - ActivitiesChanged
/// - HoldingsChanged
/// - AccountsChanged
/// - AssetsUpdated
///
/// Also carries through asset IDs from AssetsCreated when a recalc-triggering
/// event exists in the same batch.
///
/// Returns `None` if no events require portfolio recalculation.
pub fn plan_portfolio_job(events: &[DomainEvent]) -> Option<PortfolioRequestPayload> {
    let mut account_ids: HashSet<String> = HashSet::new();
    let mut asset_ids: HashSet<String> = HashSet::new();
    let mut has_recalc_events = false;

    for event in events {
        match event {
            DomainEvent::ActivitiesChanged {
                account_ids: acc_ids,
                asset_ids: a_ids,
                ..
            } => {
                has_recalc_events = true;
                account_ids.extend(acc_ids.iter().cloned());
                asset_ids.extend(a_ids.iter().cloned());
            }
            DomainEvent::HoldingsChanged {
                account_ids: acc_ids,
                asset_ids: a_ids,
            } => {
                has_recalc_events = true;
                account_ids.extend(acc_ids.iter().cloned());
                asset_ids.extend(a_ids.iter().cloned());
            }
            DomainEvent::AccountsChanged {
                account_ids: acc_ids,
                ..
            } => {
                has_recalc_events = true;
                account_ids.extend(acc_ids.iter().cloned());
            }
            DomainEvent::ManualSnapshotSaved { account_id } => {
                has_recalc_events = true;
                account_ids.insert(account_id.clone());
            }
            DomainEvent::DeviceSyncPullComplete => {
                has_recalc_events = true;
            }
            DomainEvent::AssetsUpdated { asset_ids: ids } => {
                has_recalc_events = true;
                for id in ids {
                    if !id.is_empty() {
                        asset_ids.insert(id.clone());
                    }
                }
            }
            // AssetsCreated: include IDs for sync (e.g., FX assets), but don't trigger recalc alone
            DomainEvent::AssetsCreated { asset_ids: ids } => {
                for id in ids {
                    if !id.is_empty() {
                        asset_ids.insert(id.clone());
                    }
                }
            }
            DomainEvent::AssetsMerged { .. } | DomainEvent::TrackingModeChanged { .. } => {}
        }
    }

    if !has_recalc_events {
        return None;
    }

    // Build the payload using the builder pattern
    let mut builder = PortfolioRequestPayload::builder();

    // Set account IDs if we have specific ones, otherwise None means all
    if !account_ids.is_empty() {
        builder = builder.account_ids(Some(account_ids.into_iter().collect()));
    }

    // Use incremental sync with the collected asset IDs
    let sync_mode = if asset_ids.is_empty() {
        wealthfolio_core::quotes::MarketSyncMode::Incremental { asset_ids: None }
    } else {
        wealthfolio_core::quotes::MarketSyncMode::Incremental {
            asset_ids: Some(asset_ids.into_iter().collect()),
        }
    };
    builder = builder.market_sync_mode(sync_mode);

    Some(builder.build())
}

/// Plans broker sync for eligible tracking mode changes.
///
/// Returns account IDs that need broker sync when:
/// - is_connected == true
/// - old_mode != new_mode
/// - Transition is: NOT_SET -> TRANSACTIONS/HOLDINGS or HOLDINGS -> TRANSACTIONS
pub fn plan_broker_sync(events: &[DomainEvent]) -> Vec<String> {
    let mut account_ids = Vec::new();

    for event in events {
        if let DomainEvent::TrackingModeChanged {
            account_id,
            old_mode,
            new_mode,
            is_connected,
        } = event
        {
            // Skip if not connected or mode didn't change
            if !is_connected || old_mode == new_mode {
                continue;
            }

            // Check for eligible transitions:
            // - NOT_SET -> TRANSACTIONS
            // - NOT_SET -> HOLDINGS
            // - HOLDINGS -> TRANSACTIONS
            let eligible = matches!(
                (old_mode, new_mode),
                (TrackingMode::NotSet, TrackingMode::Transactions)
                    | (TrackingMode::NotSet, TrackingMode::Holdings)
                    | (TrackingMode::Holdings, TrackingMode::Transactions)
            );

            if eligible {
                account_ids.push(account_id.clone());
            }
        }
    }

    account_ids
}

/// Plans asset enrichment for newly created assets.
///
/// Returns asset IDs from AssetsCreated events.
pub fn plan_asset_enrichment(events: &[DomainEvent]) -> Vec<String> {
    let mut asset_ids: HashSet<String> = HashSet::new();

    for event in events {
        if let DomainEvent::AssetsCreated {
            asset_ids: a_ids, ..
        } = event
        {
            asset_ids.extend(a_ids.iter().cloned());
        }
    }

    asset_ids.into_iter().collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    use wealthfolio_core::events::CurrencyChange;

    #[test]
    fn test_plan_portfolio_job_empty_events() {
        let events: Vec<DomainEvent> = vec![];
        let result = plan_portfolio_job(&events);
        assert!(result.is_none());
    }

    #[test]
    fn test_plan_portfolio_job_activities_changed() {
        let events = vec![DomainEvent::ActivitiesChanged {
            account_ids: vec!["acc1".to_string()],
            asset_ids: vec!["AAPL".to_string()],
            currencies: vec!["USD".to_string()],
        }];

        let result = plan_portfolio_job(&events);
        assert!(result.is_some());

        let payload = result.unwrap();
        assert!(payload.account_ids.is_some());
        let account_ids = payload.account_ids.unwrap();
        assert!(account_ids.contains(&"acc1".to_string()));
    }

    #[test]
    fn test_plan_portfolio_job_accounts_changed_no_fake_fx_ids() {
        let events = vec![DomainEvent::AccountsChanged {
            account_ids: vec!["acc1".to_string()],
            currency_changes: vec![CurrencyChange {
                account_id: "acc1".to_string(),
                old_currency: Some("EUR".to_string()),
                new_currency: "GBP".to_string(),
            }],
        }];

        let result = plan_portfolio_job(&events);
        assert!(result.is_some());

        let payload = result.unwrap();
        // FX assets are synced via AssetsCreated events, not constructed from currencies
        if let wealthfolio_core::quotes::MarketSyncMode::Incremental { asset_ids } =
            payload.market_sync_mode
        {
            assert!(asset_ids.is_none());
        } else {
            panic!("Expected Incremental sync mode");
        }
    }

    #[test]
    fn test_plan_portfolio_job_assets_created_contributes_ids() {
        let events = vec![
            DomainEvent::ActivitiesChanged {
                account_ids: vec!["acc1".to_string()],
                asset_ids: vec!["equity-uuid".to_string()],
                currencies: vec!["USD".to_string()],
            },
            DomainEvent::AssetsCreated {
                asset_ids: vec!["fx-uuid".to_string()],
            },
        ];

        let result = plan_portfolio_job(&events).unwrap();
        if let wealthfolio_core::quotes::MarketSyncMode::Incremental { asset_ids } =
            result.market_sync_mode
        {
            let ids = asset_ids.unwrap();
            assert!(ids.contains(&"equity-uuid".to_string()));
            assert!(ids.contains(&"fx-uuid".to_string()));
        } else {
            panic!("Expected Incremental sync mode");
        }
    }

    #[test]
    fn test_plan_portfolio_job_assets_created_no_recalc() {
        let events = vec![DomainEvent::AssetsCreated {
            asset_ids: vec!["AAPL".to_string()],
        }];

        let result = plan_portfolio_job(&events);
        assert!(result.is_none());
    }

    #[test]
    fn test_plan_portfolio_job_assets_updated_triggers_recalc() {
        let events = vec![DomainEvent::AssetsUpdated {
            asset_ids: vec!["asset-1".to_string()],
        }];

        let result = plan_portfolio_job(&events).unwrap();
        assert!(result.account_ids.is_none());

        if let wealthfolio_core::quotes::MarketSyncMode::Incremental { asset_ids } =
            result.market_sync_mode
        {
            assert_eq!(asset_ids, Some(vec!["asset-1".to_string()]));
        } else {
            panic!("Expected Incremental sync mode");
        }
    }

    #[test]
    fn test_plan_broker_sync_not_connected() {
        let events = vec![DomainEvent::TrackingModeChanged {
            account_id: "acc1".to_string(),
            old_mode: TrackingMode::NotSet,
            new_mode: TrackingMode::Transactions,
            is_connected: false, // Not connected
        }];

        let result = plan_broker_sync(&events);
        assert!(result.is_empty());
    }

    #[test]
    fn test_plan_broker_sync_eligible_transition() {
        let events = vec![DomainEvent::TrackingModeChanged {
            account_id: "acc1".to_string(),
            old_mode: TrackingMode::NotSet,
            new_mode: TrackingMode::Transactions,
            is_connected: true,
        }];

        let result = plan_broker_sync(&events);
        assert_eq!(result, vec!["acc1".to_string()]);
    }

    #[test]
    fn test_plan_broker_sync_ineligible_transition() {
        // TRANSACTIONS -> HOLDINGS is not an eligible transition
        let events = vec![DomainEvent::TrackingModeChanged {
            account_id: "acc1".to_string(),
            old_mode: TrackingMode::Transactions,
            new_mode: TrackingMode::Holdings,
            is_connected: true,
        }];

        let result = plan_broker_sync(&events);
        assert!(result.is_empty());
    }

    #[test]
    fn test_plan_asset_enrichment() {
        let events = vec![
            DomainEvent::AssetsCreated {
                asset_ids: vec!["AAPL".to_string(), "MSFT".to_string()],
            },
            DomainEvent::AssetsCreated {
                asset_ids: vec!["GOOG".to_string()],
            },
        ];

        let result = plan_asset_enrichment(&events);
        assert_eq!(result.len(), 3);
        assert!(result.contains(&"AAPL".to_string()));
        assert!(result.contains(&"MSFT".to_string()));
        assert!(result.contains(&"GOOG".to_string()));
    }

    #[test]
    fn test_plan_asset_enrichment_deduplicates() {
        let events = vec![
            DomainEvent::AssetsCreated {
                asset_ids: vec!["AAPL".to_string()],
            },
            DomainEvent::AssetsCreated {
                asset_ids: vec!["AAPL".to_string()], // Duplicate
            },
        ];

        let result = plan_asset_enrichment(&events);
        assert_eq!(result.len(), 1);
    }

    #[test]
    fn test_plan_portfolio_job_device_sync_pull_complete() {
        let events = vec![DomainEvent::device_sync_pull_complete()];

        let result = plan_portfolio_job(&events);
        assert!(result.is_some());

        let payload = result.unwrap();
        // DeviceSyncPullComplete should trigger recalculation for all accounts
        assert!(payload.account_ids.is_none());
    }

    #[test]
    fn test_plan_portfolio_job_device_sync_pull_complete_triggers_incremental_sync() {
        let events = vec![DomainEvent::device_sync_pull_complete()];

        let result = plan_portfolio_job(&events).unwrap();

        // Should use incremental sync mode
        if let wealthfolio_core::quotes::MarketSyncMode::Incremental { asset_ids } =
            result.market_sync_mode
        {
            assert!(asset_ids.is_none());
        } else {
            panic!("Expected Incremental sync mode");
        }
    }
}
```

The planner is pure logic: no I/O, no side effects. It takes a slice of events and returns a `PortfolioRequestPayload` (or `None` if nothing needs doing). Key rules:

- `AssetsCreated` alone doesn't trigger a recalc — only enrichment. A new asset with no activity doesn't affect the portfolio.
- All recalc-triggering events merge their `account_ids` and `asset_ids` into a single payload. Ten activities on two accounts = one recalc job covering those two accounts.
- The `MarketSyncMode::Incremental { asset_ids }` tells the quote service to fetch only the quotes for assets in the batch, not all assets.

## 10. The Portfolio Pipeline: Market Sync → Recalculation

Once the planner returns a payload, `run_portfolio_job` takes over. Let's see what it does.

```bash
grep -n 'async fn run_portfolio_job' /Users/sven/development/wealthfolio/apps/tauri/src/domain_events/queue_worker.rs
```

```output
177:async fn run_portfolio_job(
```

```bash
sed -n '177,320p' /Users/sven/development/wealthfolio/apps/tauri/src/domain_events/queue_worker.rs
```

```output
async fn run_portfolio_job(
    app_handle: &AppHandle,
    context: &Arc<ServiceContext>,
    payload: PortfolioRequestPayload,
) {
    let market_sync_mode = payload.market_sync_mode.clone();
    let accounts_to_recalc = payload.account_ids.clone();
    // Domain events always trigger force full recalculation
    let force_recalc = true;

    // Only perform market sync if the mode requires it
    if market_sync_mode.requires_sync() {
        let market_data_service = context.quote_service();

        // Emit sync start event
        if let Err(e) = app_handle.emit(MARKET_SYNC_START, &()) {
            error!("Failed to emit market:sync-start event: {}", e);
        }

        let sync_start = std::time::Instant::now();
        let asset_ids = market_sync_mode.asset_ids().cloned();

        // Convert MarketSyncMode to SyncMode for the quote service
        let sync_result = match market_sync_mode.to_sync_mode() {
            Some(sync_mode) => market_data_service.sync(sync_mode, asset_ids).await,
            None => {
                warn!("MarketSyncMode requires sync but returned None for SyncMode");
                Ok(wealthfolio_core::quotes::SyncResult::default())
            }
        };

        info!("Market data sync completed in: {:?}", sync_start.elapsed());

        match sync_result {
            Ok(result) => {
                let failed_syncs = result.failures;

                let health_service = context.health_service();
                health_service.clear_cache().await;

                let result_payload = MarketSyncResult { failed_syncs };
                if let Err(e) = app_handle.emit(MARKET_SYNC_COMPLETE, &result_payload) {
                    error!("Failed to emit market:sync-complete event: {}", e);
                }

                // Initialize the FxService after successful sync
                let fx_service = context.fx_service();
                if let Err(e) = fx_service.initialize() {
                    error!(
                        "Failed to initialize FxService after market data sync: {}",
                        e
                    );
                }

                // Continue to portfolio calculation
                run_portfolio_calculation(app_handle, context, accounts_to_recalc, force_recalc)
                    .await;
            }
            Err(e) => {
                if let Err(e_emit) = app_handle.emit(MARKET_SYNC_ERROR, &e.to_string()) {
                    error!("Failed to emit market:sync-error event: {}", e_emit);
                }
                error!(
                    "Market data sync failed: {}. Skipping portfolio calculation.",
                    e
                );
            }
        }
    } else {
        // MarketSyncMode::None - skip market sync, just recalculate
        debug!("Skipping market sync (MarketSyncMode::None)");
        run_portfolio_calculation(app_handle, context, accounts_to_recalc, force_recalc).await;
    }
}

/// Runs the portfolio calculation (snapshots and valuations).
async fn run_portfolio_calculation(
    app_handle: &AppHandle,
    context: &Arc<ServiceContext>,
    account_ids: Option<Vec<String>>,
    force_full_recalculation: bool,
) {
    // Emit start event
    if let Err(e) = app_handle.emit(PORTFOLIO_UPDATE_START, &()) {
        error!("Failed to emit portfolio:update-start event: {}", e);
    }

    // For TOTAL portfolio calculation, use non-archived accounts (ignores is_active)
    let accounts_for_total = match context.account_service().get_non_archived_accounts() {
        Ok(accounts) => accounts,
        Err(err) => {
            let err_msg = format!("Failed to list non-archived accounts: {}", err);
            error!("{}", err_msg);
            let _ = app_handle.emit(PORTFOLIO_UPDATE_ERROR, &err_msg);
            return;
        }
    };

    // Determine which accounts to calculate individual snapshots for:
    // - If specific account_ids provided: process those accounts (even if archived)
    // - Otherwise: process all non-archived accounts
    let mut account_ids_vec: Vec<String> = if let Some(ref target_ids) = account_ids {
        // Process the specific requested accounts (even if archived, for their own snapshots)
        target_ids.clone()
    } else {
        // No specific accounts requested - use non-archived accounts
        accounts_for_total.iter().map(|a| a.id.clone()).collect()
    };

    // Calculate holdings snapshots
    if !account_ids_vec.is_empty() {
        let ids_slice = account_ids_vec.as_slice();
        let snapshot_service = context.snapshot_service();

        let snapshot_result = if force_full_recalculation {
            snapshot_service
                .force_recalculate_holdings_snapshots(Some(ids_slice))
                .await
        } else {
            snapshot_service
                .calculate_holdings_snapshots(Some(ids_slice))
                .await
        };

        if let Err(err) = snapshot_result {
            let err_msg = format!(
                "Holdings snapshot calculation failed for targeted accounts: {}",
                err
            );
            warn!("{}", err_msg);
            let _ = app_handle.emit(PORTFOLIO_UPDATE_ERROR, &err_msg);
        }
    }

    // Calculate total portfolio snapshots
    let snapshot_service = context.snapshot_service();
    let total_result = if force_full_recalculation {
        snapshot_service
            .force_recalculate_total_portfolio_snapshots()
            .await
    } else {
        snapshot_service.calculate_total_portfolio_snapshots().await
    };
    if let Err(err) = total_result {
```

The portfolio pipeline has two phases, always in order:

**Phase 1 — Market sync**: Fetches historical quotes for the affected assets. Emits `market:sync-start` to the frontend before starting, `market:sync-complete` or `market:sync-error` when done. If sync fails, the pipeline aborts — no point recalculating with stale prices.

**Phase 2 — Portfolio calculation**: Runs holdings snapshots (per-account positions at each date) and then total portfolio valuations. The `force_full_recalculation` flag (always true when triggered by domain events) wipes and rebuilds from the first activity date rather than appending incrementally. This is expensive but ensures correctness when past activities change.

Both phases emit Tauri events to the frontend so the UI can show progress indicators.

## 11. Backend → Frontend Events

The Tauri event system is the bridge. Let's look at the event constants and how the frontend listens.

```bash
grep -n 'pub const' /Users/sven/development/wealthfolio/apps/tauri/src/events.rs | head -20
```

```output
5:pub const PORTFOLIO_TOTAL_ACCOUNT_ID: &str = "TOTAL";
8:pub const APP_READY: &str = "app:ready";
11:pub const PORTFOLIO_TRIGGER_UPDATE: &str = "portfolio:trigger-update";
14:pub const PORTFOLIO_TRIGGER_RECALCULATE: &str = "portfolio:trigger-recalculate";
17:pub const PORTFOLIO_UPDATE_START: &str = "portfolio:update-start";
20:pub const PORTFOLIO_UPDATE_COMPLETE: &str = "portfolio:update-complete";
23:pub const PORTFOLIO_UPDATE_ERROR: &str = "portfolio:update-error";
26:pub const MARKET_SYNC_START: &str = "market:sync-start";
29:pub const MARKET_SYNC_COMPLETE: &str = "market:sync-complete";
39:pub const MARKET_SYNC_ERROR: &str = "market:sync-error";
42:pub const BROKER_SYNC_START: &str = "broker:sync-start";
45:pub const BROKER_SYNC_COMPLETE: &str = "broker:sync-complete";
48:pub const BROKER_SYNC_ERROR: &str = "broker:sync-error";
```

There are two directions:

**Frontend → backend** (trigger events): `portfolio:trigger-update` and `portfolio:trigger-recalculate` — the frontend emits these to request a sync. These bypass the domain event system entirely and go straight to the `listeners.rs` handlers.

**Backend → frontend** (status events): `market:sync-start/complete/error`, `portfolio:update-start/complete/error` — emitted by `app_handle.emit()` inside the queue worker to drive the loading UI.

```bash
cat /Users/sven/development/wealthfolio/apps/tauri/src/listeners.rs
```

```output
use futures::future::join_all;
use log::{error, info, warn};
use std::sync::Arc;
use std::time::Instant;
use tauri::{async_runtime::spawn, AppHandle, Emitter, Listener, Manager};
use wealthfolio_core::constants::PORTFOLIO_TOTAL_ACCOUNT_ID;
use wealthfolio_core::health::HealthServiceTrait;
use wealthfolio_core::quotes::MarketSyncMode;

use crate::context::ServiceContext;
use crate::events::{
    emit_portfolio_trigger_recalculate, emit_portfolio_trigger_update, MarketSyncResult,
    PortfolioRequestPayload, MARKET_SYNC_COMPLETE, MARKET_SYNC_ERROR, MARKET_SYNC_START,
    PORTFOLIO_TRIGGER_RECALCULATE, PORTFOLIO_TRIGGER_UPDATE, PORTFOLIO_UPDATE_COMPLETE,
    PORTFOLIO_UPDATE_ERROR, PORTFOLIO_UPDATE_START,
};

/// Sets up the global event listeners for the application.
pub fn setup_event_listeners(handle: AppHandle) {
    // Listener for consolidated portfolio update requests
    let update_handle = handle.clone();
    handle.listen(PORTFOLIO_TRIGGER_UPDATE, move |event| {
        handle_portfolio_request(update_handle.clone(), event.payload(), false);
    });

    // Listener for full portfolio recalculation requests
    let recalc_handle = handle.clone();
    handle.listen(PORTFOLIO_TRIGGER_RECALCULATE, move |event| {
        handle_portfolio_request(recalc_handle.clone(), event.payload(), true); // force_recalc = true
    });
}

/// Handles the common logic for both portfolio update and recalculation requests.
fn handle_portfolio_request(handle: AppHandle, payload_str: &str, force_recalc: bool) {
    let event_name = if force_recalc {
        PORTFOLIO_TRIGGER_RECALCULATE
    } else {
        PORTFOLIO_TRIGGER_UPDATE
    };

    match serde_json::from_str::<PortfolioRequestPayload>(payload_str) {
        Ok(payload) => {
            let handle_clone = handle.clone(); // Clone handle for async block

            // Spawn a task to handle the update/recalculate steps
            spawn(async move {
                let market_sync_mode = payload.market_sync_mode.clone();
                let accounts_to_recalc = payload.account_ids.clone();
                let context_result = handle_clone.try_state::<Arc<ServiceContext>>();

                if let Some(context) = context_result {
                    // Only perform market sync if the mode requires it
                    if market_sync_mode.requires_sync() {
                        let market_data_service = context.quote_service();

                        // Emit sync start event
                        if let Err(e) = handle_clone.emit(MARKET_SYNC_START, &()) {
                            error!("Failed to emit market:sync-start event: {}", e);
                        }

                        let sync_start = Instant::now();
                        let asset_ids = market_sync_mode.asset_ids().cloned();

                        // Convert MarketSyncMode to SyncMode for the quote service
                        let sync_result = match market_sync_mode.to_sync_mode() {
                            Some(sync_mode) => market_data_service.sync(sync_mode, asset_ids).await,
                            None => {
                                // This shouldn't happen since we checked requires_sync()
                                warn!(
                                    "MarketSyncMode requires sync but returned None for SyncMode"
                                );
                                Ok(wealthfolio_core::quotes::SyncResult::default())
                            }
                        };

                        let sync_duration = sync_start.elapsed();
                        info!("Market data sync completed in: {:?}", sync_duration);

                        match sync_result {
                            Ok(result) => {
                                // Convert SyncResult to legacy format for backwards compatibility
                                let failed_syncs = result.failures;

                                let health_service = context.health_service();
                                let health_clone = health_service.clone();
                                spawn(async move {
                                    health_clone.clear_cache().await;
                                });

                                let result_payload = MarketSyncResult { failed_syncs };
                                if let Err(e) =
                                    handle_clone.emit(MARKET_SYNC_COMPLETE, &result_payload)
                                {
                                    error!("Failed to emit market:sync-complete event: {}", e);
                                }
                                // Initialize the FxService after successful sync
                                let fx_service = context.fx_service();
                                if let Err(e) = fx_service.initialize() {
                                    error!(
                                        "Failed to initialize FxService after market data sync: {}",
                                        e
                                    );
                                }

                                // Trigger calculation after successful sync
                                handle_portfolio_calculation(
                                    handle_clone.clone(),
                                    accounts_to_recalc,
                                    force_recalc,
                                );
                            }
                            Err(e) => {
                                if let Err(e_emit) =
                                    handle_clone.emit(MARKET_SYNC_ERROR, &e.to_string())
                                {
                                    error!("Failed to emit market:sync-error event: {}", e_emit);
                                }
                                error!("Market data sync failed: {}. Skipping portfolio calculation for this request.", e);
                            }
                        }
                    } else {
                        // MarketSyncMode::None - skip market sync, just recalculate
                        info!("Skipping market sync (MarketSyncMode::None)");
                        handle_portfolio_calculation(
                            handle_clone.clone(),
                            accounts_to_recalc,
                            force_recalc,
                        );
                    }
                } else {
                    error!(
                        "ServiceContext not found in state during market data sync for {} request.",
                        event_name
                    );
                }
            });
        }
        Err(e) => {
            error!(
                "Failed to parse payload for {}: {}. Triggering default action.",
                event_name, e
            );
            // Trigger a default action if payload parsing fails - use MarketSyncMode::None
            let fallback_payload = PortfolioRequestPayload::builder()
                .account_ids(None)
                .market_sync_mode(MarketSyncMode::None)
                .build();
            if force_recalc {
                emit_portfolio_trigger_recalculate(&handle, fallback_payload);
            } else {
                emit_portfolio_trigger_update(&handle, fallback_payload);
            }
        }
    }
}

// This function handles the portfolio snapshot and history calculation logic
fn handle_portfolio_calculation(
    app_handle: AppHandle,
    account_ids_input: Option<Vec<String>>,
    force_full_recalculation: bool,
) {
    if let Err(e) = app_handle.emit(PORTFOLIO_UPDATE_START, ()) {
        error!("Failed to emit {} event: {}", PORTFOLIO_UPDATE_START, e);
    }

    spawn(async move {
        let context = match app_handle.try_state::<Arc<ServiceContext>>() {
            Some(ctx) => ctx,
            None => {
                let err_msg =
                    "ServiceContext not found in state when triggering portfolio calculation.";
                error!("{}", err_msg);
                if let Err(e_emit) = app_handle.emit(PORTFOLIO_UPDATE_ERROR, err_msg) {
                    error!(
                        "Failed to emit {} event: {}",
                        PORTFOLIO_UPDATE_ERROR, e_emit
                    );
                }
                return;
            }
        };

        let account_service = context.account_service();
        let snapshot_service = context.snapshot_service();
        let valuation_service = context.valuation_service();

        // Step 0: Resolve initially targeted active accounts for individual calculations.
        // This list might be empty if account_ids_input is None and no accounts are active,
        // or if account_ids_input specified accounts that are now all inactive.
        let initially_targeted_active_accounts: Vec<String> =
            match account_service.list_accounts(Some(true), None, account_ids_input.as_deref()) {
                Ok(accounts) => accounts.iter().map(|a| a.id.clone()).collect(),
                Err(e) => {
                    let err_msg = format!("Failed to list active accounts: {}", e);
                    error!("{}", err_msg);
                    if let Err(e_emit) = app_handle.emit(PORTFOLIO_UPDATE_ERROR, &err_msg) {
                        error!(
                            "Failed to emit {} event: {}",
                            PORTFOLIO_UPDATE_ERROR, e_emit
                        );
                    }
                    return;
                }
            };

        // --- Step 1: Calculate Account-Specific Snapshots (only if there are specific active accounts to process) ---
        if !initially_targeted_active_accounts.is_empty() {
            let account_snapshot_result = if force_full_recalculation {
                snapshot_service
                    .force_recalculate_holdings_snapshots(Some(
                        initially_targeted_active_accounts.as_slice(),
                    ))
                    .await
            } else {
                snapshot_service
                    .calculate_holdings_snapshots(Some(
                        initially_targeted_active_accounts.as_slice(),
                    ))
                    .await
            };

            if let Err(e) = account_snapshot_result {
                let err_msg = format!(
                    "calculate_holdings_snapshots for targeted accounts failed: {}",
                    e
                );
                error!("{}", err_msg);
                if let Err(e_emit) = app_handle.emit(PORTFOLIO_UPDATE_ERROR, &err_msg) {
                    error!(
                        "Failed to emit {} event: {}",
                        PORTFOLIO_UPDATE_ERROR, e_emit
                    );
                }
            }
        }

        // --- Step 2: Calculate TOTAL portfolio snapshot ---
        let total_result = if force_full_recalculation {
            snapshot_service
                .force_recalculate_total_portfolio_snapshots()
                .await
        } else {
            snapshot_service.calculate_total_portfolio_snapshots().await
        };
        if let Err(e) = total_result {
            let err_msg = format!("Failed to calculate TOTAL portfolio snapshot: {}", e);
            error!("{}", err_msg);
            if let Err(e_emit) = app_handle.emit(PORTFOLIO_UPDATE_ERROR, &err_msg) {
                error!(
                    "Failed to emit {} event: {}",
                    PORTFOLIO_UPDATE_ERROR, e_emit
                );
            }
            return;
        }

        // --- Step 2.5: Update position status from TOTAL snapshot ---
        // This derives open/closed position transitions for quote sync planning
        if let Ok(Some(total_snapshot)) =
            snapshot_service.get_latest_holdings_snapshot(PORTFOLIO_TOTAL_ACCOUNT_ID)
        {
            let quote_service = context.quote_service();

            // Extract asset quantities from the TOTAL snapshot
            let current_holdings: std::collections::HashMap<String, rust_decimal::Decimal> =
                total_snapshot
                    .positions
                    .iter()
                    .map(|(asset_id, position)| (asset_id.clone(), position.quantity))
                    .collect();

            if let Err(e) = quote_service
                .update_position_status_from_holdings(&current_holdings)
                .await
            {
                warn!(
                    "Failed to update position status from holdings: {}. Quote sync planning may be affected.",
                    e
                );
            }
        }

        // --- Step 3: Calculate Valuation History ---
        let mut accounts_for_valuation = initially_targeted_active_accounts;
        if !accounts_for_valuation.contains(&PORTFOLIO_TOTAL_ACCOUNT_ID.to_string()) {
            accounts_for_valuation.push(PORTFOLIO_TOTAL_ACCOUNT_ID.to_string());
        }

        if !accounts_for_valuation.is_empty() {
            let history_futures = accounts_for_valuation.iter().map(|account_id| {
                let valuation_service_clone = valuation_service.clone();
                let account_id_clone = account_id.clone();
                async move {
                    let result = valuation_service_clone
                        .calculate_valuation_history(&account_id_clone, force_full_recalculation)
                        .await;
                    (account_id_clone, result)
                }
            });

            let history_results = join_all(history_futures).await;

            let mut history_errors: Vec<String> = Vec::new();
            for (account_id, result) in history_results {
                if let Err(e) = result {
                    let err_detail = format!("Account '{}': {}", account_id, e);
                    error!("Failed to calculate valuation history: {}", err_detail);
                    history_errors.push(err_detail);
                }
            }

            if !history_errors.is_empty() {
                error!(
                    "Valuation history calculation completed with errors: {}",
                    history_errors.join("; ")
                );
            }
        }

        if let Err(e) = app_handle.emit(PORTFOLIO_UPDATE_COMPLETE, ()) {
            error!("Failed to emit {} event: {}", PORTFOLIO_UPDATE_COMPLETE, e);
        }
    });
}
```

The portfolio calculation runs in three sequential steps, each logged and each allowed to fail independently:

1. **Account snapshots** — builds daily holdings positions per account (`force_recalculate_holdings_snapshots`). This is the expensive step: for each account, walk all activities from inception, apply splits and FX, compute end-of-day positions.
2. **Total portfolio snapshot** — same thing for the synthetic `TOTAL` account aggregated across all real accounts.
3. **Valuation history** — runs concurrently (`join_all`) for all affected accounts: takes the snapshots + quote prices and produces the daily portfolio value series used for charts.

After all three complete, `PORTFOLIO_UPDATE_COMPLETE` is emitted to the frontend.

## 12. Frontend: Sync Context and Cache Invalidation

The frontend listens for these backend events using Tauri's `listen` API. The `PortfolioSyncContext` is the central place that reacts.

```bash
cat /Users/sven/development/wealthfolio/apps/frontend/src/context/portfolio-sync-context.tsx
```

```output
import { createContext, useContext, useState, useCallback, useMemo, type ReactNode } from "react";

export type SyncStatus = "idle" | "syncing-market" | "calculating-portfolio";

interface PortfolioSyncContextType {
  status: SyncStatus;
  message: string;
  setMarketSyncing: () => void;
  setPortfolioCalculating: () => void;
  setIdle: () => void;
}

const PortfolioSyncContext = createContext<PortfolioSyncContextType | undefined>(undefined);

const STATUS_MESSAGES: Record<SyncStatus, string> = {
  idle: "",
  "syncing-market": "Syncing market data...",
  "calculating-portfolio": "Calculating portfolio...",
};

export function PortfolioSyncProvider({ children }: { children: ReactNode }) {
  const [status, setStatus] = useState<SyncStatus>("idle");

  const setMarketSyncing = useCallback(() => {
    setStatus("syncing-market");
  }, []);

  const setPortfolioCalculating = useCallback(() => {
    setStatus("calculating-portfolio");
  }, []);

  const setIdle = useCallback(() => {
    setStatus("idle");
  }, []);

  const value = useMemo<PortfolioSyncContextType>(
    () => ({
      status,
      message: STATUS_MESSAGES[status],
      setMarketSyncing,
      setPortfolioCalculating,
      setIdle,
    }),
    [status, setMarketSyncing, setPortfolioCalculating, setIdle],
  );

  return <PortfolioSyncContext.Provider value={value}>{children}</PortfolioSyncContext.Provider>;
}

export function usePortfolioSync() {
  const context = useContext(PortfolioSyncContext);
  if (!context) {
    throw new Error("usePortfolioSync must be used within a PortfolioSyncProvider");
  }
  return context;
}

// Optional hook that returns null if used outside provider (for places where provider might not exist)
export function usePortfolioSyncOptional() {
  return useContext(PortfolioSyncContext);
}
```

```bash
cat /Users/sven/development/wealthfolio/apps/frontend/src/adapters/tauri/events.ts
```

```output
// Event Listeners
import type {
  EventCallback as TauriEventCallback,
  UnlistenFn as TauriUnlistenFn,
} from "@tauri-apps/api/event";
import { listen } from "@tauri-apps/api/event";

import type { EventCallback, UnlistenFn } from "../types";

// Helper to adapt Tauri's event callback to our unified type
const adaptCallback = <T>(handler: EventCallback<T>): TauriEventCallback<T> => {
  return (event) => handler({ event: event.event, payload: event.payload, id: event.id });
};

// Helper to adapt Tauri's unlisten function to our unified type
const adaptUnlisten = (unlisten: TauriUnlistenFn): UnlistenFn => {
  return async () => unlisten();
};

export const listenFileDropHover = async <T>(handler: EventCallback<T>): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("tauri://file-drop-hover", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export const listenFileDrop = async <T>(handler: EventCallback<T>): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("tauri://file-drop", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export const listenFileDropCancelled = async <T>(
  handler: EventCallback<T>,
): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("tauri://file-drop-cancelled", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export const listenPortfolioUpdateStart = async <T>(
  handler: EventCallback<T>,
): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("portfolio:update-start", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export const listenPortfolioUpdateComplete = async <T>(
  handler: EventCallback<T>,
): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("portfolio:update-complete", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export const listenDatabaseRestored = async <T>(handler: EventCallback<T>): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("database-restored", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export const listenPortfolioUpdateError = async <T>(
  handler: EventCallback<T>,
): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("portfolio:update-error", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};

export async function listenMarketSyncComplete<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("market:sync-complete", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export async function listenMarketSyncStart<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("market:sync-start", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export async function listenMarketSyncError<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("market:sync-error", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export async function listenBrokerSyncStart<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("broker:sync-start", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export async function listenBrokerSyncComplete<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("broker:sync-complete", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export async function listenBrokerSyncError<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("broker:sync-error", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export async function listenNavigateToRoute<T>(handler: EventCallback<T>): Promise<UnlistenFn> {
  const unlisten = await listen<T>("navigate-to-route", adaptCallback(handler));
  return adaptUnlisten(unlisten);
}

export const listenDeepLink = async <T>(handler: EventCallback<T>): Promise<UnlistenFn> => {
  const unlisten = await listen<T>("deep-link-received", adaptCallback(handler));
  return adaptUnlisten(unlisten);
};
```

```bash
cat /Users/sven/development/wealthfolio/apps/frontend/src/use-global-event-listener.ts
```

```output
// useGlobalEventListener.ts
import {
  updatePortfolio,
  listenMarketSyncComplete,
  listenMarketSyncError,
  listenMarketSyncStart,
  listenPortfolioUpdateComplete,
  listenPortfolioUpdateError,
  listenPortfolioUpdateStart,
} from "@/adapters";
import { usePortfolioSyncOptional } from "@/context/portfolio-sync-context";
import { useIsMobileViewport } from "@/hooks/use-platform";
import { useQueryClient } from "@tanstack/react-query";
import { useEffect, useRef } from "react";
import { useNavigate } from "react-router-dom";
import { toast } from "sonner";
import {
  isDesktop,
  listenBrokerSyncComplete,
  listenBrokerSyncError,
  listenDatabaseRestored,
  logger,
} from "@/adapters";

const TOAST_IDS = {
  marketSyncStart: "market-sync-start",
  portfolioUpdateStart: "portfolio-update-start",
  portfolioUpdateError: "portfolio-update-error",
  brokerSyncStart: "broker-sync-start",
} as const;

const useGlobalEventListener = () => {
  const queryClient = useQueryClient();
  const navigate = useNavigate();
  const hasTriggeredInitialUpdate = useRef(false);
  const isDesktopEnv = isDesktop;
  const isMobileViewport = useIsMobileViewport();
  const syncContext = usePortfolioSyncOptional();

  // Use refs to avoid stale closures in event handlers
  const isMobileViewportRef = useRef(isMobileViewport);
  const syncContextRef = useRef(syncContext);
  const queryClientRef = useRef(queryClient);
  const navigateRef = useRef(navigate);

  // Keep refs up to date
  useEffect(() => {
    isMobileViewportRef.current = isMobileViewport;
    syncContextRef.current = syncContext;
    queryClientRef.current = queryClient;
    navigateRef.current = navigate;
  });

  useEffect(() => {
    let isMounted = true;
    let cleanupFn: (() => void) | undefined;

    const handleMarketSyncStart = () => {
      if (isMobileViewportRef.current && syncContextRef.current) {
        syncContextRef.current.setMarketSyncing();
      } else {
        toast.loading("Syncing market data...", {
          id: TOAST_IDS.marketSyncStart,
          duration: 3000,
        });
      }
    };

    const handleMarketSyncComplete = (event: { payload: { failed_syncs: [string, string][] } }) => {
      const { failed_syncs } = event.payload || { failed_syncs: [] };

      if (isMobileViewportRef.current && syncContextRef.current) {
        syncContextRef.current.setIdle();
      } else {
        toast.dismiss(TOAST_IDS.marketSyncStart);
      }

      // Show error toast on both mobile and desktop for failed syncs
      if (failed_syncs && failed_syncs.length > 0) {
        const failedSymbols = failed_syncs.map(([symbol]) => symbol).join(", ");
        toast.error("Market Data Update Incomplete", {
          id: `market-sync-error-${failedSymbols || "unknown"}`,
          description: `Unable to update market data for: ${failedSymbols}. This may affect your portfolio calculations and analytics. Please try again later.`,
          duration: 15000,
          action: {
            label: "View Securities",
            onClick: () => {
              navigateRef.current("/settings/securities");
            },
          },
          classNames: {
            toast: "!flex-wrap",
            content: "!flex-[1_0_calc(100%-2rem)]",
            actionButton: "!ml-auto",
          },
        });
      }
    };

    const handleMarketSyncError = (event: { payload: string }) => {
      const errorMsg = event.payload || "Unknown error";
      if (isMobileViewportRef.current && syncContextRef.current) {
        syncContextRef.current.setIdle();
      } else {
        toast.dismiss(TOAST_IDS.marketSyncStart);
      }
      toast.error("Market Data Sync Failed", {
        description: `${errorMsg}. Please try again later.`,
        duration: 10000,
      });
      logger.error("Market sync error: " + errorMsg);
    };

    const handlePortfolioUpdateStart = () => {
      if (isMobileViewportRef.current && syncContextRef.current) {
        syncContextRef.current.setPortfolioCalculating();
      } else {
        toast.loading("Calculating portfolio performance...", {
          id: TOAST_IDS.portfolioUpdateStart,
          duration: 2000,
        });
      }
    };

    const handlePortfolioUpdateError = (error: string) => {
      if (isMobileViewportRef.current && syncContextRef.current) {
        syncContextRef.current.setIdle();
      } else {
        toast.dismiss(TOAST_IDS.portfolioUpdateStart);
      }
      toast.error("Portfolio Update Failed", {
        id: TOAST_IDS.portfolioUpdateError,
        description:
          "There was an error updating your portfolio. Please try again or contact support if the issue persists.",
        duration: 5000,
      });
      logger.error("Portfolio Update Error: " + error);
    };

    const handlePortfolioUpdateComplete = () => {
      if (isMobileViewportRef.current && syncContextRef.current) {
        syncContextRef.current.setIdle();
      } else {
        toast.dismiss(TOAST_IDS.portfolioUpdateStart);
      }
      queryClientRef.current.invalidateQueries();
    };

    const handleDatabaseRestored = () => {
      queryClientRef.current.invalidateQueries();
      toast.success("Database restored successfully", {
        description: "Please restart the application to ensure all data is properly refreshed.",
      });
    };

    const handleBrokerSyncComplete = (event: {
      payload: {
        success: boolean;
        message: string;
        accountsSynced?: { created: number; updated: number; skipped: number };
        activitiesSynced?: { activitiesUpserted: number; assetsInserted: number };
        holdingsSynced?: {
          accountsSynced: number;
          snapshotsUpserted: number;
          positionsUpserted: number;
          assetsInserted: number;
          newAssetIds: string[];
        };
        newAccounts?: {
          localAccountId: string;
          providerAccountId: string;
          defaultName: string;
          currency: string;
          institutionName?: string;
        }[];
      };
    }) => {
      const { success, message, accountsSynced, activitiesSynced, holdingsSynced, newAccounts } =
        event.payload || {
          success: false,
          message: "Unknown error",
        };

      // Dismiss the loading toast
      toast.dismiss(TOAST_IDS.brokerSyncStart);

      // Invalidate queries that could be affected by sync
      queryClientRef.current.invalidateQueries();

      if (success) {
        // Check if there are new accounts that need configuration
        if (newAccounts && newAccounts.length > 0) {
          toast.info("New accounts found", {
            description: `${newAccounts.length} new account(s) need to be configured`,
            action: {
              label: "Review",
              onClick: () => {
                navigateRef.current("/settings/accounts");
              },
            },
            duration: Infinity, // Don't auto-dismiss - user must act or dismiss manually
          });
        } else {
          // Build description with key numbers
          const accountsCreated = accountsSynced?.created ?? 0;
          const accountsUpdated = accountsSynced?.updated ?? 0;
          const activities = activitiesSynced?.activitiesUpserted ?? 0;
          const activityAssets = activitiesSynced?.assetsInserted ?? 0;
          const positions = holdingsSynced?.positionsUpserted ?? 0;
          const holdingsAccounts = holdingsSynced?.accountsSynced ?? 0;
          const holdingsAssets = holdingsSynced?.assetsInserted ?? 0;
          const totalNewAssets = activityAssets + holdingsAssets;

          const hasChanges =
            accountsCreated > 0 ||
            accountsUpdated > 0 ||
            activities > 0 ||
            totalNewAssets > 0 ||
            positions > 0;

          let description: string;
          if (hasChanges) {
            const parts: string[] = [];
            if (accountsCreated > 0) parts.push(`${accountsCreated} new accounts`);
            if (accountsUpdated > 0) parts.push(`${accountsUpdated} accounts updated`);
            if (activities > 0) parts.push(`${activities} activities`);
            if (positions > 0) parts.push(`${positions} positions (${holdingsAccounts} accounts)`);
            if (totalNewAssets > 0) parts.push(`${totalNewAssets} new assets`);
            description = parts.join(" · ");
          } else {
            description = "Everything is up to date";
          }

          toast.success("Broker Sync Complete", {
            description,
            duration: 5000,
          });
        }
      } else {
        toast.error("Broker Sync Failed", {
          description: message,
          duration: 10000,
        });
      }
    };

    const handleBrokerSyncError = (event: { payload: { error: string } }) => {
      const { error } = event.payload || { error: "Unknown error" };
      // Dismiss the loading toast
      toast.dismiss(TOAST_IDS.brokerSyncStart);
      toast.error("Broker Sync Failed", {
        description: error,
        duration: 10000,
      });
    };

    const setupListeners = async () => {
      const unlistenPortfolioSyncStart = await listenPortfolioUpdateStart(
        handlePortfolioUpdateStart,
      );
      const unlistenPortfolioSyncComplete = await listenPortfolioUpdateComplete(
        handlePortfolioUpdateComplete,
      );
      const unlistenPortfolioSyncError = await listenPortfolioUpdateError((event) => {
        handlePortfolioUpdateError(event.payload as string);
      });
      const unlistenMarketStart = await listenMarketSyncStart(handleMarketSyncStart);
      const unlistenMarketComplete = await listenMarketSyncComplete(handleMarketSyncComplete);
      const unlistenMarketError = await listenMarketSyncError(handleMarketSyncError);
      const unlistenDatabaseRestored = await listenDatabaseRestored(handleDatabaseRestored);
      const unlistenBrokerSyncComplete = await listenBrokerSyncComplete(handleBrokerSyncComplete);
      const unlistenBrokerSyncError = await listenBrokerSyncError(handleBrokerSyncError);

      const cleanup = () => {
        unlistenPortfolioSyncStart();
        unlistenPortfolioSyncComplete();
        unlistenPortfolioSyncError();
        unlistenMarketStart();
        unlistenMarketComplete();
        unlistenMarketError();
        unlistenDatabaseRestored();
        unlistenBrokerSyncComplete();
        unlistenBrokerSyncError();
      };

      // If unmounted while setting up, clean up immediately
      if (!isMounted) {
        cleanup();
        return;
      }

      cleanupFn = cleanup;

      // Trigger initial portfolio update after listeners are set up
      if (!hasTriggeredInitialUpdate.current) {
        hasTriggeredInitialUpdate.current = true;
        logger.debug("Triggering initial portfolio update from frontend");

        // Trigger portfolio update
        updatePortfolio().catch((error) => {
          logger.error("Failed to trigger initial portfolio update: " + String(error));
        });
        // Note: Update check is now handled by useCheckUpdateOnStartup query in UpdateDialog
      }
    };

    setupListeners().catch((error) => {
      console.error("Failed to setup global event listeners:", error);
    });

    return () => {
      isMounted = false;
      cleanupFn?.();
    };
  }, [isDesktopEnv]); // Only re-run if isDesktopEnv changes (which it won't)

  return null;
};

export default useGlobalEventListener;
```

`useGlobalEventListener` is a hook that lives at the app root and never unmounts. It wires up all the backend→frontend event handlers in one place:

- `market:sync-start` → show toast (desktop) or set sync context status (mobile)
- `market:sync-complete` → dismiss toast, show errors for failed symbols
- `portfolio:update-start` → show calculating toast/status
- `portfolio:update-complete` → dismiss toast, then **`queryClient.invalidateQueries()`**
- `portfolio:update-error` → show error toast

The `invalidateQueries()` call with no arguments is the key: it marks every cached query as stale. React Query then refetches any query whose component is currently mounted. Pages don't subscribe to events — they just declare their data needs with `useQuery`, and React Query handles the rest.

The hook also triggers an initial `updatePortfolio()` call once all listeners are registered, driving the first sync after mount.

## 13. Tying It Together: A Page Component

Let's look at how a real page consumes all of this. The holdings page is a good example.

```bash
sed -n '1,80p' /Users/sven/development/wealthfolio/apps/frontend/src/pages/holdings/holdings-page.tsx
```

```output
import { Button } from "@wealthfolio/ui/components/ui/button";
import { Icons } from "@wealthfolio/ui/components/ui/icons";
import { EmptyPlaceholder } from "@wealthfolio/ui";
import { useCallback, useMemo, useState } from "react";
import { useNavigate, useSearchParams } from "react-router-dom";

import { SwipablePage, SwipablePageView } from "@/components/page";
import { AccountSelector } from "@/components/account-selector";
import { ActionPalette, type ActionPaletteGroup } from "@/components/action-palette";
import { useAccounts } from "@/hooks/use-accounts";
import { useHoldings } from "@/hooks/use-holdings";
import {
  useAlternativeHoldings,
  useDeleteAlternativeAsset,
  useLinkLiability,
  useUnlinkLiability,
} from "@/hooks/use-alternative-assets";
import { usePersistentState } from "@/hooks/use-persistent-state";
import {
  PORTFOLIO_ACCOUNT_ID,
  HOLDING_CATEGORY_FILTERS,
  apiKindToAlternativeAssetKind,
} from "@/lib/constants";
import { Account, HoldingType, AlternativeAssetHolding, AlternativeAssetKind } from "@/lib/types";
import { canAddHoldings } from "@/lib/activity-restrictions";
import { useIsMobileViewport } from "@/hooks/use-platform";
import { HoldingsMobileFilterSheet } from "./components/holdings-mobile-filter-sheet";
import { HoldingsTable } from "./components/holdings-table";
import { HoldingsTableMobile } from "./components/holdings-table-mobile";
import { AlternativeHoldingsTable } from "./components/alternative-holdings-table";
import { AlternativeHoldingsListMobile } from "./components/alternative-holdings-list-mobile";
import { HoldingsEditMode } from "./components/holdings-edit-mode";
import {
  AlternativeAssetQuickAddModal,
  AssetDetailsSheet,
  UpdateValuationModal,
  type AssetDetailsSheetAsset,
  type LinkableAsset,
  type LinkedLiability,
} from "@/pages/asset/alternative-assets";
import { updateAlternativeAssetMetadata } from "@/adapters";
import { ClassificationSheet } from "@/components/classification/classification-sheet";
import { useUpdatePortfolioMutation } from "@/hooks/use-calculate-portfolio";
import { useQueryClient } from "@tanstack/react-query";
import { QueryKeys } from "@/lib/query-keys";
import { useSettingsContext } from "@/lib/settings-provider";

export const HoldingsPage = () => {
  const isMobileViewport = useIsMobileViewport();
  const navigate = useNavigate();
  const [searchParams] = useSearchParams();
  const currentTab = searchParams.get("tab") ?? "investments";
  const queryClient = useQueryClient();
  const { settings } = useSettingsContext();
  const baseCurrency = settings?.baseCurrency ?? "USD";

  const [selectedAccount, setSelectedAccount] = useState<Account | null>({
    id: PORTFOLIO_ACCOUNT_ID,
    name: "All Portfolio",
    accountType: "PORTFOLIO" as unknown as Account["accountType"],
    balance: 0,
    currency: baseCurrency,
    isDefault: false,
    isActive: true,
    createdAt: new Date(),
    updatedAt: new Date(),
  } as Account);

  const { holdings, isLoading } = useHoldings(selectedAccount?.id ?? PORTFOLIO_ACCOUNT_ID);
  const { accounts, isLoading: isAccountsLoading } = useAccounts();
  const { data: alternativeHoldings, isLoading: isAlternativeHoldingsLoading } =
    useAlternativeHoldings();

  // Mobile filter state
  const [selectedTypes, setSelectedTypes] = useState<string[]>([]);
  const [isFilterSheetOpen, setIsFilterSheetOpen] = useState(false);
  const [isAlternativeAssetModalOpen, setIsAlternativeAssetModalOpen] = useState(false);
  const [sortBy, setSortBy] = usePersistentState<"symbol" | "marketValue">(
    "holdings-sort-by",
    "marketValue",
```

```bash
grep -n 'useHoldings\|useQuery\|queryFn\|QueryKeys' /Users/sven/development/wealthfolio/apps/frontend/src/hooks/use-holdings.ts | head -20
```

```output
1:import { useQuery } from "@tanstack/react-query";
4:import { QueryKeys } from "@/lib/query-keys";
6:export function useHoldings(accountId: string) {
12:  } = useQuery<Holding[], Error>({
13:    queryKey: [QueryKeys.HOLDINGS, accountId],
14:    queryFn: () => getHoldings(accountId),
```

```bash
cat /Users/sven/development/wealthfolio/apps/frontend/src/hooks/use-holdings.ts
```

```output
import { useQuery } from "@tanstack/react-query";
import { Holding } from "@/lib/types";
import { getHoldings } from "@/adapters";
import { QueryKeys } from "@/lib/query-keys";

export function useHoldings(accountId: string) {
  const {
    data: holdings = [],
    isLoading,
    isError,
    error,
  } = useQuery<Holding[], Error>({
    queryKey: [QueryKeys.HOLDINGS, accountId],
    queryFn: () => getHoldings(accountId),
    enabled: !!accountId,
  });

  return { holdings, isLoading, isError, error };
}
```

The page delegates all data fetching to purpose-built hooks like `useHoldings`. Each hook is a thin wrapper around React Query's `useQuery`. The query key `[QueryKeys.HOLDINGS, accountId]` is the cache key — when `invalidateQueries()` fires after `portfolio:update-complete`, React Query sees that any mounted component with a matching key needs a refresh. `getHoldings` is the shared adapter function which calls `invoke("get_holdings", { accountId })`.

The page itself has no idea a sync just happened. It just renders whatever React Query gives it — loading skeletons while fetching, data when ready.

## 14. The Complete Loop

Here is the full flow from start to finish, as one sequence:

App starts:
  → desktop_setup() runs synchronously
  → SQLite pool + write actor created
  → 15+ repositories instantiated with Arc<Pool> and WriteHandle
  → 15+ services instantiated with Arc<dyn Trait> repos
  → TauriDomainEventSink created → returns (sink, receiver)
  → sink injected into every service that mutates data
  → ServiceContext assembled, managed by Tauri state
  → queue worker spawned as tokio task with the receiver
  → listeners registered for portfolio:trigger-* events
  → app:ready emitted to frontend
  → frontend mounts, useGlobalEventListener registers event handlers
  → updatePortfolio() called, emits portfolio:trigger-update
  → listeners.rs handler fires, spawns tokio task
  → market sync runs via quote_service.sync()
  → market:sync-start emitted, frontend toast appears
  → quotes fetched from Yahoo Finance, stored in SQLite via write actor
  → market:sync-complete emitted, frontend toast updates
  → portfolio calculation runs in three steps:
      1. force_recalculate_holdings_snapshots() per account
      2. force_recalculate_total_portfolio_snapshots()
      3. calculate_valuation_history() for all accounts in parallel
  → portfolio:update-complete emitted
  → handlePortfolioUpdateComplete() fires in frontend
  → queryClient.invalidateQueries() marks all cached data stale
  → useHoldings, useValuation, and all active hooks refetch
  → pages re-render with live data

User creates account:
  → form submit calls createAccount(newAccount) from shared adapter
  → invoke("create_account", { account }) sent over Tauri IPC
  → commands::account::create_account() receives it
  → account_service.create_account() runs:
      - fx_service.register_currency_pair() if currency != base
      - repository.create() calls writer.exec_tx()
      - write actor runs Diesel INSERT inside a transaction
      - event_sink.emit(AccountsChanged { ... }) sends to mpsc channel
  → Account returned to frontend, React Query updates that mutation's cache
  → queue worker receives AccountsChanged
  → 1 second debounce window collects any more events
  → planner.plan_portfolio_job([AccountsChanged]) returns PortfolioRequestPayload
  → run_portfolio_job() runs: market sync then recalculation
  → portfolio:update-complete emitted
  → queryClient.invalidateQueries() fires
  → all mounted queries refetch, UI refreshes automatically

## 15. Key Architectural Invariants

A few design choices that explain why the code looks the way it does:

**Commands are always thin.** Commands never contain business logic — they exist only to deserialize IPC input and call a service. This keeps the Tauri crate as a thin platform adapter; the actual logic lives in crates that could be reused with a different shell.

**Services never know about the database.** A service holds an `Arc<dyn SomeRepositoryTrait>`. Concrete SQLite types live only in `storage-sqlite` and `providers.rs`. This boundary is enforced by Rust's module system.

**Mutations always end with an event.** Every write operation emits a domain event. There is no code path that writes to the database without notifying the queue worker. This is why the comment says "Domain events handle recalculation automatically" — you don't have to remember to trigger an update, the system does it.

**The write actor prevents data races.** SQLite supports one writer at a time. Rather than holding a mutex for the duration of every write, the write actor serializes operations through a channel. Reads use a connection pool (r2d2) and happen concurrently. Writes are serialized but async — callers await the result without blocking other tasks.

**React Query is the only client-side cache.** There is no Redux, no Zustand, no custom store. All server state is owned by React Query. Invalidation is the only update mechanism: when the backend signals completion, the frontend tells React Query to throw away everything and re-ask. This is simple and correct, at the cost of extra requests after each sync.
