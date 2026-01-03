# FPL Squad Optimizer - Rust Implementation Design

> **Rust-Specific Design Document**
> 
> This document maps the language-agnostic specification to Rust patterns, crates, and implementation details. It includes incremental feature rollout, configuration, logging, Docker deployment, and result persistence.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Crate Dependencies](#2-crate-dependencies)
3. [Core Data Types](#3-core-data-types)
4. [Module Design](#4-module-design)
5. [Feature Flags & Incremental Rollout](#5-feature-flags--incremental-rollout)
6. [Configuration System](#6-configuration-system)
7. [Logging & Observability](#7-logging--observability)
8. [Result Persistence & History](#8-result-persistence--history)
9. [Docker Deployment](#9-docker-deployment)
10. [Error Handling](#10-error-handling)
11. [Testing Strategy](#11-testing-strategy)
12. [Performance Considerations](#12-performance-considerations)

---

## 1. Project Structure

```
fpl-optimizer/
├── Cargo.toml                    # Workspace manifest
├── Cargo.lock
├── Dockerfile
├── docker-compose.yml
├── config/
│   ├── default.json              # Default configuration
│   └── example.json              # Example user config
├── data/
│   └── .gitkeep                  # Projection files go here
├── crates/
│   ├── fpl-core/                 # Core data types & traits
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── player.rs
│   │       ├── squad.rs
│   │       ├── solution.rs
│   │       ├── config.rs
│   │       └── error.rs
│   ├── fpl-api/                  # FPL API client
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── client.rs
│   │       ├── models.rs
│   │       └── cache.rs
│   ├── fpl-data/                 # Data parsing & processing
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── parser.rs
│   │       ├── merger.rs
│   │       └── filter.rs
│   ├── fpl-solver/               # Optimization model & solver
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── model.rs
│   │       ├── variables.rs
│   │       ├── constraints.rs
│   │       ├── objective.rs
│   │       ├── mps.rs
│   │       └── solution.rs
│   ├── fpl-storage/              # Result persistence
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── sqlite.rs
│   │       ├── json.rs
│   │       └── models.rs
│   └── fpl-cli/                  # CLI application
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs
│           ├── commands/
│           │   ├── mod.rs
│           │   ├── solve.rs
│           │   ├── simulate.rs
│           │   ├── history.rs
│           │   └── config.rs
│           └── output.rs
└── tests/
    ├── integration/
    └── fixtures/
```

### Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/fpl-core",
    "crates/fpl-api",
    "crates/fpl-data",
    "crates/fpl-solver",
    "crates/fpl-storage",
    "crates/fpl-cli",
]

[workspace.package]
version = "0.1.0"
edition = "2021"
rust-version = "1.75"
license = "MIT OR Apache-2.0"
repository = "https://github.com/username/fpl-optimizer"

[workspace.dependencies]
# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Async runtime
tokio = { version = "1.35", features = ["full"] }

# HTTP client
reqwest = { version = "0.11", features = ["json", "rustls-tls"], default-features = false }

# Data processing
polars = { version = "0.36", features = ["lazy", "csv", "json"] }

# Optimization
highs = "1.6"
good_lp = { version = "1.8", features = ["highs"] }

# CLI
clap = { version = "4.4", features = ["derive", "env"] }

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }

# Error handling
thiserror = "1.0"
anyhow = "1.0"

# Database
rusqlite = { version = "0.30", features = ["bundled"] }

# Time
chrono = { version = "0.4", features = ["serde"] }

# Testing
pretty_assertions = "1.4"
```

---

## 2. Crate Dependencies

### By Crate

#### fpl-core

```toml
[package]
name = "fpl-core"
version.workspace = true
edition.workspace = true

[dependencies]
serde = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
```

#### fpl-api

```toml
[package]
name = "fpl-api"
version.workspace = true
edition.workspace = true

[dependencies]
fpl-core = { path = "../fpl-core" }
reqwest = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

[dev-dependencies]
wiremock = "0.5"
tokio-test = "0.4"
```

#### fpl-data

```toml
[package]
name = "fpl-data"
version.workspace = true
edition.workspace = true

[dependencies]
fpl-core = { path = "../fpl-core" }
fpl-api = { path = "../fpl-api" }
polars = { workspace = true }
serde = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }
```

#### fpl-solver

```toml
[package]
name = "fpl-solver"
version.workspace = true
edition.workspace = true

[features]
default = ["highs"]
highs = ["dep:highs"]
good_lp = ["dep:good_lp"]

[dependencies]
fpl-core = { path = "../fpl-core" }
highs = { workspace = true, optional = true }
good_lp = { workspace = true, optional = true }
thiserror = { workspace = true }
tracing = { workspace = true }
```

#### fpl-storage

```toml
[package]
name = "fpl-storage"
version.workspace = true
edition.workspace = true

[features]
default = ["sqlite"]
sqlite = ["dep:rusqlite"]
json-store = []

[dependencies]
fpl-core = { path = "../fpl-core" }
rusqlite = { workspace = true, optional = true }
serde = { workspace = true }
serde_json = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }
```

#### fpl-cli

```toml
[package]
name = "fpl-cli"
version.workspace = true
edition.workspace = true

[[bin]]
name = "fpl"
path = "src/main.rs"

[dependencies]
fpl-core = { path = "../fpl-core" }
fpl-api = { path = "../fpl-api" }
fpl-data = { path = "../fpl-data" }
fpl-solver = { path = "../fpl-solver" }
fpl-storage = { path = "../fpl-storage" }
clap = { workspace = true }
tokio = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
anyhow = { workspace = true }
serde_json = { workspace = true }
directories = "5.0"
indicatif = "0.17"  # Progress bars
comfy-table = "7.1" # Pretty tables
```

---

## 3. Core Data Types

### fpl-core/src/player.rs

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// Player position in FPL
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Position {
    #[serde(alias = "G", alias = "GKP", alias = "1")]
    Goalkeeper,
    #[serde(alias = "D", alias = "DEF", alias = "2")]
    Defender,
    #[serde(alias = "M", alias = "MID", alias = "3")]
    Midfielder,
    #[serde(alias = "F", alias = "FWD", alias = "4")]
    Forward,
}

impl Position {
    pub fn as_index(&self) -> usize {
        match self {
            Position::Goalkeeper => 1,
            Position::Defender => 2,
            Position::Midfielder => 3,
            Position::Forward => 4,
        }
    }
    
    pub fn squad_required(&self) -> usize {
        match self {
            Position::Goalkeeper => 2,
            Position::Defender => 5,
            Position::Midfielder => 5,
            Position::Forward => 3,
        }
    }
    
    pub fn lineup_min(&self) -> usize {
        match self {
            Position::Goalkeeper => 1,
            Position::Defender => 3,
            Position::Midfielder => 2,
            Position::Forward => 1,
        }
    }
    
    pub fn lineup_max(&self) -> usize {
        match self {
            Position::Goalkeeper => 1,
            Position::Defender => 5,
            Position::Midfielder => 5,
            Position::Forward => 3,
        }
    }
}

/// Gameweek-specific player projection
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Projection {
    pub expected_points: f64,
    pub expected_minutes: f64,
}

/// A player with all relevant data
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Player {
    pub id: u32,
    pub name: String,
    pub full_name: Option<String>,
    pub position: Position,
    pub team_id: u8,
    pub team_name: String,
    /// Current price in £0.1M (e.g., 130 = £13.0M)
    pub price: u16,
    /// Per-gameweek projections
    pub projections: HashMap<u8, Projection>,
}

impl Player {
    /// Price in millions (e.g., 13.0)
    pub fn price_millions(&self) -> f64 {
        self.price as f64 / 10.0
    }
    
    /// Total expected points across all projected gameweeks
    pub fn total_expected_points(&self) -> f64 {
        self.projections.values().map(|p| p.expected_points).sum()
    }
    
    /// Total expected minutes across all projected gameweeks
    pub fn total_expected_minutes(&self) -> f64 {
        self.projections.values().map(|p| p.expected_minutes).sum()
    }
}

/// Player as part of a squad (with ownership info)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OwnedPlayer {
    pub player_id: u32,
    /// Price when purchased (in £0.1M)
    pub purchase_price: u16,
    /// Current selling price (in £0.1M)
    pub selling_price: u16,
}

impl OwnedPlayer {
    /// Calculate selling price based on FPL rules
    /// Selling price = purchase + floor((current - purchase) / 2)
    pub fn calculate_selling_price(purchase_price: u16, current_price: u16) -> u16 {
        if current_price > purchase_price {
            purchase_price + (current_price - purchase_price) / 2
        } else {
            current_price
        }
    }
}
```

### fpl-core/src/squad.rs

```rust
use serde::{Deserialize, Serialize};
use crate::player::OwnedPlayer;

/// Available chips
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Chip {
    Wildcard,
    FreeHit,
    BenchBoost,
    TripleCaptain,
}

impl Chip {
    pub fn short_name(&self) -> &'static str {
        match self {
            Chip::Wildcard => "WC",
            Chip::FreeHit => "FH",
            Chip::BenchBoost => "BB",
            Chip::TripleCaptain => "TC",
        }
    }
}

/// Chip availability status
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ChipStatus {
    pub wildcard_available: bool,
    pub free_hit_available: bool,
    pub bench_boost_available: bool,
    pub triple_captain_available: bool,
}

/// Record of chip usage
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChipUsage {
    pub chip: Chip,
    pub gameweek: u8,
}

/// Current squad state
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SquadState {
    /// Players in squad (exactly 15)
    pub players: Vec<OwnedPlayer>,
    /// Money in bank (in £0.1M)
    pub bank: u16,
    /// Available free transfers (1-5)
    pub free_transfers: u8,
    /// Chips available
    pub chips_available: ChipStatus,
    /// Chips already used this season
    pub chips_used: Vec<ChipUsage>,
}

impl SquadState {
    pub fn bank_millions(&self) -> f64 {
        self.bank as f64 / 10.0
    }
    
    pub fn total_value(&self) -> u16 {
        self.players.iter().map(|p| p.selling_price).sum::<u16>() + self.bank
    }
}

/// Transfer details
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Transfer {
    pub player_in: u32,
    pub player_out: u32,
    pub gameweek: u8,
    pub cost: u8,  // 0 for free, 4 for hit
}
```

### fpl-core/src/solution.rs

```rust
use serde::{Deserialize, Serialize};
use crate::squad::Chip;
use std::collections::HashMap;

/// A player selection in a gameweek
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlayerPick {
    pub player_id: u32,
    pub name: String,
    pub position: crate::player::Position,
    pub team: String,
    pub expected_points: f64,
    pub expected_minutes: f64,
    pub is_captain: bool,
    pub is_vice_captain: bool,
    pub is_lineup: bool,
    pub bench_position: Option<u8>,  // 0-3 if on bench
    pub is_transfer_in: bool,
    pub is_transfer_out: bool,
    pub buy_price: Option<f64>,
    pub sell_price: Option<f64>,
}

impl PlayerPick {
    pub fn multiplier(&self) -> u8 {
        if self.is_captain { 2 } else if self.is_lineup { 1 } else { 0 }
    }
    
    pub fn expected_contribution(&self) -> f64 {
        self.expected_points * self.multiplier() as f64
    }
}

/// Plan for a single gameweek
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GameweekPlan {
    pub gameweek: u8,
    pub picks: Vec<PlayerPick>,
    pub chip_used: Option<Chip>,
    pub free_transfers: u8,
    pub transfers_made: u8,
    pub hits_taken: u8,
    pub bank: f64,
    pub expected_points: f64,
}

impl GameweekPlan {
    pub fn lineup(&self) -> impl Iterator<Item = &PlayerPick> {
        self.picks.iter().filter(|p| p.is_lineup)
    }
    
    pub fn bench(&self) -> impl Iterator<Item = &PlayerPick> {
        self.picks.iter().filter(|p| p.bench_position.is_some())
    }
    
    pub fn transfers_in(&self) -> impl Iterator<Item = &PlayerPick> {
        self.picks.iter().filter(|p| p.is_transfer_in)
    }
    
    pub fn transfers_out(&self) -> impl Iterator<Item = &PlayerPick> {
        self.picks.iter().filter(|p| p.is_transfer_out)
    }
    
    pub fn captain(&self) -> Option<&PlayerPick> {
        self.picks.iter().find(|p| p.is_captain)
    }
    
    pub fn vice_captain(&self) -> Option<&PlayerPick> {
        self.picks.iter().find(|p| p.is_vice_captain)
    }
}

/// Complete solution from optimizer
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Solution {
    pub id: String,  // UUID
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub iteration: u32,
    pub score: f64,
    pub gameweeks: Vec<GameweekPlan>,
    pub config_hash: String,  // Hash of config used
}

impl Solution {
    pub fn total_expected_points(&self) -> f64 {
        self.gameweeks.iter().map(|gw| gw.expected_points).sum()
    }
    
    pub fn total_hits(&self) -> u8 {
        self.gameweeks.iter().map(|gw| gw.hits_taken).sum()
    }
    
    pub fn summary(&self) -> String {
        let mut lines = Vec::new();
        for gw in &self.gameweeks {
            let transfers: Vec<_> = gw.transfers_in()
                .zip(gw.transfers_out())
                .map(|(i, o)| format!("{} → {}", o.name, i.name))
                .collect();
            
            let transfer_str = if transfers.is_empty() {
                "Roll".to_string()
            } else {
                transfers.join(", ")
            };
            
            let chip_str = gw.chip_used
                .map(|c| format!("({}) ", c.short_name()))
                .unwrap_or_default();
            
            lines.push(format!("GW{}: {}{}", gw.gameweek, chip_str, transfer_str));
        }
        lines.join("\n")
    }
}
```

### fpl-core/src/config.rs

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::path::PathBuf;

/// Main configuration structure
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct Config {
    // === Planning ===
    /// Number of gameweeks to optimize
    pub horizon: u8,
    
    // === Objective Tuning ===
    /// Decay rate for future gameweeks (0.9 = 10% discount per GW)
    pub decay_base: f64,
    /// Base value of a free transfer in points
    pub ft_value: f64,
    /// Per-state FT values (key: "2", "3", "4", "5")
    pub ft_value_list: HashMap<String, f64>,
    /// Bench position weights (key: "0", "1", "2", "3")
    pub bench_weights: HashMap<String, f64>,
    /// Vice-captain weight (for tiebreaking)
    pub vcap_weight: f64,
    /// Value of £1M in bank
    pub itb_value: f64,
    /// Points deducted per hit
    pub hit_cost: u8,
    
    // === Data Source ===
    /// Projection data source filename (without .csv)
    pub datasource: String,
    /// Weights for multiple data sources
    pub data_weights: HashMap<String, f64>,
    /// Path to data directory
    pub data_dir: Option<PathBuf>,
    
    // === Team Data ===
    /// How to get team data: "id", "json", "file"
    pub team_data: TeamDataSource,
    /// FPL team ID (when team_data = "id")
    pub team_id: Option<u32>,
    /// Override the next gameweek
    pub override_next_gw: Option<u8>,
    
    // === Player Constraints ===
    /// Player IDs to exclude entirely
    pub banned: Vec<u32>,
    /// Player IDs to always include
    pub locked: Vec<u32>,
    /// Player IDs to keep during filtering
    pub keep: Vec<u32>,
    
    // === Transfer Constraints ===
    /// GWs at end of horizon with no transfers
    pub no_transfer_last_gws: u8,
    /// Maximum total hits across horizon
    pub hit_limit: Option<u8>,
    /// Maximum hits per gameweek
    pub weekly_hit_limit: Option<u8>,
    /// Booked transfers
    pub booked_transfers: Vec<BookedTransfer>,
    
    // === Chip Scheduling ===
    /// Gameweeks to use Wildcard
    pub use_wc: Vec<u8>,
    /// Gameweeks to use Bench Boost
    pub use_bb: Vec<u8>,
    /// Gameweeks to use Free Hit
    pub use_fh: Vec<u8>,
    /// Gameweeks to use Triple Captain
    pub use_tc: Vec<u8>,
    /// Chip usage limits
    pub chip_limits: ChipLimits,
    
    // === Player Filtering ===
    /// Minimum expected minutes to include player
    pub xmin_lb: f64,
    /// Percentile cutoff for EV/price filtering
    pub ev_per_price_cutoff: f64,
    /// Always keep top N% by expected value
    pub keep_top_ev_percent: f64,
    
    // === Solver Settings ===
    /// Solver backend: "highs", "cbc", "gurobi"
    pub solver: String,
    /// Time limit in seconds
    pub time_limit_secs: u64,
    /// Optimality gap tolerance (0.0 = optimal)
    pub optimality_gap: f64,
    /// Random seed for solver
    pub random_seed: Option<u64>,
    
    // === Output Settings ===
    /// Number of alternative solutions to generate
    pub num_iterations: u32,
    /// Print squad details to console
    pub print_squads: bool,
    /// Print transfer summary
    pub print_transfer_summary: bool,
    /// Export timeline image
    pub export_image: bool,
    /// Verbose solver output
    pub verbose: bool,
    
    // === Storage Settings ===
    /// Path to results database
    pub database_path: Option<PathBuf>,
    /// Keep last N runs in history
    pub history_limit: Option<u32>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum TeamDataSource {
    Id,
    Json,
    File,
}

impl Default for TeamDataSource {
    fn default() -> Self {
        TeamDataSource::Id
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BookedTransfer {
    pub gw: u8,
    pub transfer_in: Option<u32>,
    pub transfer_out: Option<u32>,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ChipLimits {
    pub wc: u8,
    pub bb: u8,
    pub fh: u8,
    pub tc: u8,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            horizon: 8,
            decay_base: 0.9,
            ft_value: 1.5,
            ft_value_list: [
                ("2".into(), 2.0),
                ("3".into(), 1.6),
                ("4".into(), 1.3),
                ("5".into(), 1.1),
            ].into_iter().collect(),
            bench_weights: [
                ("0".into(), 0.03),
                ("1".into(), 0.21),
                ("2".into(), 0.06),
                ("3".into(), 0.002),
            ].into_iter().collect(),
            vcap_weight: 0.1,
            itb_value: 0.08,
            hit_cost: 4,
            datasource: "solio".into(),
            data_weights: HashMap::new(),
            data_dir: None,
            team_data: TeamDataSource::Id,
            team_id: None,
            override_next_gw: None,
            banned: Vec::new(),
            locked: Vec::new(),
            keep: Vec::new(),
            no_transfer_last_gws: 2,
            hit_limit: None,
            weekly_hit_limit: Some(0),
            booked_transfers: Vec::new(),
            use_wc: Vec::new(),
            use_bb: Vec::new(),
            use_fh: Vec::new(),
            use_tc: Vec::new(),
            chip_limits: ChipLimits::default(),
            xmin_lb: 300.0,
            ev_per_price_cutoff: 30.0,
            keep_top_ev_percent: 5.0,
            solver: "highs".into(),
            time_limit_secs: 600,
            optimality_gap: 0.0,
            random_seed: None,
            num_iterations: 1,
            print_squads: true,
            print_transfer_summary: true,
            export_image: false,
            verbose: false,
            database_path: None,
            history_limit: Some(100),
        }
    }
}

impl Config {
    /// Load config from file, merging with defaults
    pub fn from_file(path: &std::path::Path) -> Result<Self, ConfigError> {
        let content = std::fs::read_to_string(path)?;
        let config: Self = serde_json::from_str(&content)?;
        Ok(config)
    }
    
    /// Merge another config into this one (other takes precedence)
    pub fn merge(&mut self, other: PartialConfig) {
        if let Some(v) = other.horizon { self.horizon = v; }
        if let Some(v) = other.decay_base { self.decay_base = v; }
        // ... (implement for all fields)
    }
    
    /// Generate a hash of the config for identification
    pub fn hash(&self) -> String {
        use std::collections::hash_map::DefaultHasher;
        use std::hash::{Hash, Hasher};
        
        let json = serde_json::to_string(self).unwrap_or_default();
        let mut hasher = DefaultHasher::new();
        json.hash(&mut hasher);
        format!("{:016x}", hasher.finish())
    }
}

/// Partial config for merging/overrides
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct PartialConfig {
    pub horizon: Option<u8>,
    pub decay_base: Option<f64>,
    pub team_id: Option<u32>,
    // ... (all fields as Option<T>)
}

#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("Failed to read config file: {0}")]
    IoError(#[from] std::io::Error),
    #[error("Failed to parse config: {0}")]
    ParseError(#[from] serde_json::Error),
    #[error("Invalid config: {0}")]
    ValidationError(String),
}
```

---

## 4. Module Design

### 4.1 fpl-api Module

```rust
// fpl-api/src/client.rs

use reqwest::Client;
use std::sync::Arc;
use tokio::sync::RwLock;
use crate::cache::Cache;
use crate::models::*;

const BASE_URL: &str = "https://fantasy.premierleague.com/api";
const CACHE_TTL_SECS: u64 = 300;  // 5 minutes

pub struct FplClient {
    client: Client,
    cache: Arc<RwLock<Cache>>,
}

impl FplClient {
    pub fn new() -> Self {
        Self {
            client: Client::builder()
                .user_agent("fpl-optimizer/0.1")
                .timeout(std::time::Duration::from_secs(30))
                .build()
                .expect("Failed to create HTTP client"),
            cache: Arc::new(RwLock::new(Cache::new(CACHE_TTL_SECS))),
        }
    }
    
    /// Get bootstrap-static data (all players, teams, events)
    #[tracing::instrument(skip(self))]
    pub async fn get_bootstrap(&self) -> Result<BootstrapData, ApiError> {
        let url = format!("{}/bootstrap-static/", BASE_URL);
        self.get_cached(&url).await
    }
    
    /// Get team data for a specific team ID
    #[tracing::instrument(skip(self))]
    pub async fn get_team(&self, team_id: u32) -> Result<TeamData, ApiError> {
        let url = format!("{}/entry/{}/", BASE_URL, team_id);
        self.get_cached(&url).await
    }
    
    /// Get current squad picks
    #[tracing::instrument(skip(self))]
    pub async fn get_picks(&self, team_id: u32, gameweek: u8) -> Result<PicksData, ApiError> {
        let url = format!("{}/entry/{}/event/{}/picks/", BASE_URL, team_id, gameweek);
        self.get_cached(&url).await
    }
    
    /// Get transfer history
    #[tracing::instrument(skip(self))]
    pub async fn get_transfers(&self, team_id: u32) -> Result<Vec<TransferRecord>, ApiError> {
        let url = format!("{}/entry/{}/transfers/", BASE_URL, team_id);
        self.get_cached(&url).await
    }
    
    /// Get fixtures
    #[tracing::instrument(skip(self))]
    pub async fn get_fixtures(&self) -> Result<Vec<Fixture>, ApiError> {
        let url = format!("{}/fixtures/", BASE_URL);
        self.get_cached(&url).await
    }
    
    async fn get_cached<T: serde::de::DeserializeOwned>(&self, url: &str) -> Result<T, ApiError> {
        // Check cache first
        {
            let cache = self.cache.read().await;
            if let Some(data) = cache.get(url) {
                tracing::debug!("Cache hit for {}", url);
                return serde_json::from_str(&data).map_err(ApiError::Parse);
            }
        }
        
        // Fetch from API
        tracing::debug!("Fetching {}", url);
        let response = self.client.get(url).send().await?;
        
        if !response.status().is_success() {
            return Err(ApiError::HttpError(response.status()));
        }
        
        let text = response.text().await?;
        
        // Store in cache
        {
            let mut cache = self.cache.write().await;
            cache.set(url.to_string(), text.clone());
        }
        
        serde_json::from_str(&text).map_err(ApiError::Parse)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("HTTP request failed: {0}")]
    Request(#[from] reqwest::Error),
    #[error("HTTP error: {0}")]
    HttpError(reqwest::StatusCode),
    #[error("Failed to parse response: {0}")]
    Parse(serde_json::Error),
}
```

### 4.2 fpl-solver Module

```rust
// fpl-solver/src/model.rs

use fpl_core::{Player, SquadState, Config, Solution};
use std::collections::HashMap;

/// Optimization model builder
pub struct FplModel {
    players: Vec<Player>,
    initial_state: SquadState,
    config: Config,
    next_gw: u8,
    
    // Variable indices
    var_squad: HashMap<(u32, u8), usize>,      // (player_id, gw) -> var index
    var_lineup: HashMap<(u32, u8), usize>,
    var_captain: HashMap<(u32, u8), usize>,
    var_bench: HashMap<(u32, u8, u8), usize>,  // (player_id, gw, bench_pos)
    var_transfer_in: HashMap<(u32, u8), usize>,
    var_transfer_out: HashMap<(u32, u8), usize>,
    var_chips: HashMap<(Chip, u8), usize>,
    
    // MPS data
    rows: Vec<Row>,
    columns: Vec<Column>,
    bounds: Vec<Bound>,
    objective: Vec<(usize, f64)>,
}

struct Row {
    name: String,
    row_type: RowType,
    rhs: f64,
}

enum RowType {
    Objective,
    LessEqual,
    GreaterEqual,
    Equal,
}

struct Column {
    name: String,
    coefficients: Vec<(usize, f64)>,  // (row_index, coefficient)
    is_binary: bool,
}

struct Bound {
    var_index: usize,
    lower: f64,
    upper: f64,
}

impl FplModel {
    pub fn new(
        players: Vec<Player>,
        initial_state: SquadState,
        config: Config,
        next_gw: u8,
    ) -> Self {
        Self {
            players,
            initial_state,
            config,
            next_gw,
            var_squad: HashMap::new(),
            var_lineup: HashMap::new(),
            var_captain: HashMap::new(),
            var_bench: HashMap::new(),
            var_transfer_in: HashMap::new(),
            var_transfer_out: HashMap::new(),
            var_chips: HashMap::new(),
            rows: Vec::new(),
            columns: Vec::new(),
            bounds: Vec::new(),
            objective: Vec::new(),
        }
    }
    
    /// Build the complete optimization model
    #[tracing::instrument(skip(self))]
    pub fn build(&mut self) -> Result<(), ModelError> {
        tracing::info!("Building optimization model");
        
        self.create_variables()?;
        self.add_squad_constraints()?;
        self.add_formation_constraints()?;
        self.add_team_limit_constraints()?;
        self.add_transfer_constraints()?;
        self.add_chip_constraints()?;
        self.add_optional_constraints()?;
        self.set_objective()?;
        
        tracing::info!(
            "Model built: {} variables, {} constraints",
            self.columns.len(),
            self.rows.len()
        );
        
        Ok(())
    }
    
    /// Export model to MPS format
    pub fn export_mps(&self, path: &std::path::Path) -> Result<(), std::io::Error> {
        use std::io::Write;
        
        let mut file = std::fs::File::create(path)?;
        
        writeln!(file, "NAME          fpl_optimizer")?;
        
        // ROWS section
        writeln!(file, "ROWS")?;
        for (i, row) in self.rows.iter().enumerate() {
            let type_char = match row.row_type {
                RowType::Objective => 'N',
                RowType::LessEqual => 'L',
                RowType::GreaterEqual => 'G',
                RowType::Equal => 'E',
            };
            writeln!(file, " {}  {}", type_char, row.name)?;
        }
        
        // COLUMNS section
        writeln!(file, "COLUMNS")?;
        let mut in_integer_section = false;
        
        for col in &self.columns {
            if col.is_binary && !in_integer_section {
                writeln!(file, "    MARKER    'MARKER'    'INTORG'")?;
                in_integer_section = true;
            } else if !col.is_binary && in_integer_section {
                writeln!(file, "    MARKER    'MARKER'    'INTEND'")?;
                in_integer_section = false;
            }
            
            for (row_idx, coef) in &col.coefficients {
                writeln!(file, "    {}    {}    {}", 
                    col.name, self.rows[*row_idx].name, coef)?;
            }
        }
        
        if in_integer_section {
            writeln!(file, "    MARKER    'MARKER'    'INTEND'")?;
        }
        
        // RHS section
        writeln!(file, "RHS")?;
        for row in &self.rows {
            if row.rhs != 0.0 {
                writeln!(file, "    RHS    {}    {}", row.name, row.rhs)?;
            }
        }
        
        // BOUNDS section
        writeln!(file, "BOUNDS")?;
        for bound in &self.bounds {
            let col = &self.columns[bound.var_index];
            if col.is_binary {
                writeln!(file, " BV BND    {}", col.name)?;
            } else {
                if bound.lower != 0.0 {
                    writeln!(file, " LO BND    {}    {}", col.name, bound.lower)?;
                }
                if bound.upper != f64::INFINITY {
                    writeln!(file, " UP BND    {}    {}", col.name, bound.upper)?;
                }
            }
        }
        
        writeln!(file, "ENDATA")?;
        
        Ok(())
    }
    
    /// Solve using HiGHS
    #[cfg(feature = "highs")]
    pub fn solve(&self) -> Result<Solution, SolverError> {
        use highs::{HighsModel, Sense};
        
        // Export to temp file
        let temp_path = std::env::temp_dir().join("fpl_model.mps");
        self.export_mps(&temp_path)?;
        
        // Create and configure solver
        let mut model = HighsModel::default();
        model.read_model(temp_path.to_str().unwrap())?;
        
        model.set_option("time_limit", self.config.time_limit_secs as f64);
        model.set_option("mip_rel_gap", self.config.optimality_gap);
        model.set_option("log_to_console", self.config.verbose);
        
        if let Some(seed) = self.config.random_seed {
            model.set_option("random_seed", seed as i32);
        }
        
        // Solve
        let solved = model.solve();
        
        // Extract solution
        self.extract_solution(&solved)
    }
    
    // ... (constraint building methods)
}

#[derive(Debug, thiserror::Error)]
pub enum ModelError {
    #[error("Invalid model state: {0}")]
    InvalidState(String),
}

#[derive(Debug, thiserror::Error)]
pub enum SolverError {
    #[error("Failed to export model: {0}")]
    ExportError(#[from] std::io::Error),
    #[error("Solver failed: {0}")]
    SolverFailed(String),
    #[error("No solution found")]
    NoSolution,
}
```

---

## 5. Feature Flags & Incremental Rollout

### Feature Roadmap

```
Phase 1: Core MVP (v0.1)
├── Basic squad optimization
├── Single gameweek
├── JSON config loading
├── Console output
└── No chips

Phase 2: Multi-GW (v0.2)
├── Multi-period optimization
├── Transfer planning
├── Free transfer tracking
└── Hit penalty

Phase 3: Chips (v0.3)
├── Wildcard support
├── Free Hit support
├── Bench Boost support
├── Triple Captain support
└── Chip scheduling

Phase 4: Advanced (v0.4)
├── Multiple iterations
├── Sensitivity analysis
├── Result persistence
└── History queries

Phase 5: Polish (v0.5)
├── Timeline visualization
├── Docker support
├── REST API (optional)
└── Web UI (optional)
```

### Cargo Feature Flags

```toml
# fpl-solver/Cargo.toml
[features]
default = ["highs", "chips"]

# Solver backends
highs = ["dep:highs"]
cbc = ["dep:coin_cbc"]
gurobi = ["dep:grb"]

# Functionality
chips = []              # Enable chip support
multi-period = []       # Enable multi-GW optimization
iterations = []         # Enable multiple solution iterations
simulations = []        # Enable Monte Carlo simulations

# Output
visualization = ["dep:plotters"]
json-output = []

# Storage
sqlite = ["dep:rusqlite"]
postgres = ["dep:tokio-postgres"]
```

### Conditional Compilation

```rust
// In solver code
impl FplModel {
    fn add_chip_constraints(&mut self) -> Result<(), ModelError> {
        #[cfg(not(feature = "chips"))]
        {
            tracing::debug!("Chips feature disabled, skipping chip constraints");
            return Ok(());
        }
        
        #[cfg(feature = "chips")]
        {
            // Add wildcard constraints
            // Add free hit constraints
            // Add bench boost constraints
            // Add triple captain constraints
        }
    }
}
```

---

## 6. Configuration System

### Config Loading Priority

```rust
// fpl-cli/src/config.rs

use fpl_core::{Config, PartialConfig};
use std::path::PathBuf;

/// Configuration loader with priority handling
pub struct ConfigLoader {
    /// Base config (embedded defaults)
    base: Config,
    /// Paths to config files (in priority order)
    config_files: Vec<PathBuf>,
    /// CLI overrides
    cli_overrides: PartialConfig,
    /// Environment variable prefix
    env_prefix: String,
}

impl ConfigLoader {
    pub fn new() -> Self {
        Self {
            base: Config::default(),
            config_files: Vec::new(),
            cli_overrides: PartialConfig::default(),
            env_prefix: "FPL".to_string(),
        }
    }
    
    /// Add a config file to load (later files override earlier)
    pub fn add_file(mut self, path: PathBuf) -> Self {
        self.config_files.push(path);
        self
    }
    
    /// Add CLI argument overrides
    pub fn with_cli_args(mut self, args: &CliArgs) -> Self {
        self.cli_overrides = args.to_partial_config();
        self
    }
    
    /// Load and merge all configuration sources
    pub fn load(self) -> Result<Config, ConfigError> {
        let mut config = self.base;
        
        // 1. Load from files (in order)
        for path in &self.config_files {
            if path.exists() {
                tracing::info!("Loading config from {:?}", path);
                let file_config = Config::from_file(path)?;
                config = merge_configs(config, file_config);
            } else {
                tracing::warn!("Config file not found: {:?}", path);
            }
        }
        
        // 2. Apply environment variables
        config = self.apply_env_vars(config);
        
        // 3. Apply CLI overrides (highest priority)
        config.merge(self.cli_overrides);
        
        // 4. Validate
        config.validate()?;
        
        Ok(config)
    }
    
    fn apply_env_vars(&self, mut config: Config) -> Config {
        // FPL_TEAM_ID -> team_id
        if let Ok(val) = std::env::var(format!("{}_TEAM_ID", self.env_prefix)) {
            if let Ok(id) = val.parse() {
                config.team_id = Some(id);
            }
        }
        
        // FPL_HORIZON -> horizon
        if let Ok(val) = std::env::var(format!("{}_HORIZON", self.env_prefix)) {
            if let Ok(h) = val.parse() {
                config.horizon = h;
            }
        }
        
        // ... more env vars
        
        config
    }
}
```

### Example Config File

```json
// config/user.json
{
    "team_id": 12345,
    "horizon": 8,
    "decay_base": 0.9,
    "datasource": "solio",
    
    "banned": [100, 200],
    "locked": [427],
    
    "use_wc": [],
    "use_bb": [25],
    "use_fh": [],
    "use_tc": [],
    
    "solver": "highs",
    "time_limit_secs": 300,
    "verbose": true,
    
    "database_path": "./data/history.db",
    "history_limit": 50
}
```

---

## 7. Logging & Observability

### Logging Setup

```rust
// fpl-cli/src/main.rs

use tracing_subscriber::{fmt, prelude::*, EnvFilter, Layer};

fn setup_logging(verbose: bool, json_logs: bool, log_file: Option<&Path>) {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| {
            if verbose {
                EnvFilter::new("fpl=debug,info")
            } else {
                EnvFilter::new("fpl=info,warn")
            }
        });
    
    let subscriber = tracing_subscriber::registry();
    
    // Console layer
    let console_layer = if json_logs {
        fmt::layer()
            .json()
            .with_target(true)
            .with_file(true)
            .with_line_number(true)
            .boxed()
    } else {
        fmt::layer()
            .with_target(false)
            .with_level(true)
            .compact()
            .boxed()
    };
    
    let subscriber = subscriber.with(console_layer.with_filter(env_filter.clone()));
    
    // File layer (if specified)
    if let Some(log_path) = log_file {
        let file = std::fs::File::create(log_path).expect("Failed to create log file");
        let file_layer = fmt::layer()
            .json()
            .with_writer(file)
            .with_filter(env_filter);
        subscriber.with(file_layer).init();
    } else {
        subscriber.init();
    }
}
```

### Structured Logging

```rust
// Throughout the codebase
use tracing::{info, debug, warn, error, instrument, span, Level};

impl FplModel {
    #[instrument(skip(self), fields(players = self.players.len()))]
    pub fn build(&mut self) -> Result<(), ModelError> {
        let span = span!(Level::INFO, "model_build");
        let _enter = span.enter();
        
        info!("Starting model build");
        
        debug!(
            horizon = self.config.horizon,
            decay = self.config.decay_base,
            "Configuration loaded"
        );
        
        let start = std::time::Instant::now();
        
        self.create_variables()?;
        info!(variables = self.columns.len(), "Variables created");
        
        self.add_constraints()?;
        info!(constraints = self.rows.len(), "Constraints added");
        
        info!(
            elapsed_ms = start.elapsed().as_millis(),
            "Model build complete"
        );
        
        Ok(())
    }
}
```

### Log Output Examples

**Human-readable (default):**
```
2024-01-15T10:23:45.123Z INFO  Starting FPL optimizer
2024-01-15T10:23:45.456Z INFO  Loading config from config/user.json
2024-01-15T10:23:45.789Z INFO  Fetching bootstrap data from FPL API
2024-01-15T10:23:46.012Z DEBUG Cache miss, fetching from network
2024-01-15T10:23:47.234Z INFO  Loaded 650 players
2024-01-15T10:23:47.567Z INFO  Building optimization model
2024-01-15T10:23:48.890Z INFO  Model built: 15234 variables, 48921 constraints
2024-01-15T10:23:49.123Z INFO  Solving with HiGHS...
2024-01-15T10:24:15.456Z INFO  Solution found: 152.34 expected points
```

**JSON (for aggregation):**
```json
{"timestamp":"2024-01-15T10:23:45.123Z","level":"INFO","target":"fpl_cli","message":"Starting FPL optimizer","version":"0.1.0"}
{"timestamp":"2024-01-15T10:23:47.567Z","level":"INFO","target":"fpl_solver::model","message":"Building optimization model","players":650,"horizon":8}
{"timestamp":"2024-01-15T10:24:15.456Z","level":"INFO","target":"fpl_solver","message":"Solution found","score":152.34,"elapsed_secs":26.3}
```

---

## 8. Result Persistence & History

### Database Schema

```sql
-- SQLite schema for result storage

CREATE TABLE IF NOT EXISTS runs (
    id TEXT PRIMARY KEY,                    -- UUID
    created_at TEXT NOT NULL,               -- ISO8601 timestamp
    config_hash TEXT NOT NULL,              -- Hash of config used
    config_json TEXT NOT NULL,              -- Full config as JSON
    team_id INTEGER,
    next_gameweek INTEGER NOT NULL,
    horizon INTEGER NOT NULL,
    status TEXT NOT NULL,                   -- 'running', 'completed', 'failed'
    error_message TEXT,
    solve_time_secs REAL,
    
    UNIQUE(config_hash, created_at)
);

CREATE TABLE IF NOT EXISTS solutions (
    id TEXT PRIMARY KEY,                    -- UUID
    run_id TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    iteration INTEGER NOT NULL,
    score REAL NOT NULL,
    total_expected_points REAL NOT NULL,
    total_hits INTEGER NOT NULL,
    summary_json TEXT NOT NULL,             -- Serialized Solution
    
    UNIQUE(run_id, iteration)
);

CREATE TABLE IF NOT EXISTS gameweek_plans (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    solution_id TEXT NOT NULL REFERENCES solutions(id) ON DELETE CASCADE,
    gameweek INTEGER NOT NULL,
    expected_points REAL NOT NULL,
    bank REAL NOT NULL,
    free_transfers INTEGER NOT NULL,
    transfers_made INTEGER NOT NULL,
    hits_taken INTEGER NOT NULL,
    chip_used TEXT,                         -- NULL, 'WC', 'FH', 'BB', 'TC'
    picks_json TEXT NOT NULL,               -- Serialized Vec<PlayerPick>
    
    UNIQUE(solution_id, gameweek)
);

CREATE INDEX idx_runs_created_at ON runs(created_at DESC);
CREATE INDEX idx_runs_team_id ON runs(team_id);
CREATE INDEX idx_solutions_score ON solutions(score DESC);
```

### Storage Trait & Implementations

```rust
// fpl-storage/src/lib.rs

use async_trait::async_trait;
use fpl_core::{Config, Solution};

/// Storage backend trait
#[async_trait]
pub trait Storage: Send + Sync {
    /// Save a new run
    async fn save_run(&self, run: &Run) -> Result<(), StorageError>;
    
    /// Update run status
    async fn update_run_status(&self, run_id: &str, status: RunStatus) -> Result<(), StorageError>;
    
    /// Save a solution
    async fn save_solution(&self, solution: &Solution, run_id: &str) -> Result<(), StorageError>;
    
    /// Get recent runs
    async fn get_recent_runs(&self, limit: usize) -> Result<Vec<Run>, StorageError>;
    
    /// Get solutions for a run
    async fn get_solutions(&self, run_id: &str) -> Result<Vec<Solution>, StorageError>;
    
    /// Get best solution by score
    async fn get_best_solution(&self, run_id: &str) -> Result<Option<Solution>, StorageError>;
    
    /// Search runs by criteria
    async fn search_runs(&self, query: RunQuery) -> Result<Vec<Run>, StorageError>;
    
    /// Delete old runs (keep last N)
    async fn prune_history(&self, keep_count: usize) -> Result<usize, StorageError>;
}

#[derive(Debug, Clone)]
pub struct Run {
    pub id: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub config_hash: String,
    pub config: Config,
    pub team_id: Option<u32>,
    pub next_gameweek: u8,
    pub horizon: u8,
    pub status: RunStatus,
    pub error_message: Option<String>,
    pub solve_time_secs: Option<f64>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RunStatus {
    Running,
    Completed,
    Failed,
}

#[derive(Debug, Default)]
pub struct RunQuery {
    pub team_id: Option<u32>,
    pub gameweek: Option<u8>,
    pub status: Option<RunStatus>,
    pub from_date: Option<chrono::DateTime<chrono::Utc>>,
    pub to_date: Option<chrono::DateTime<chrono::Utc>>,
    pub limit: Option<usize>,
}

// SQLite implementation
#[cfg(feature = "sqlite")]
pub mod sqlite;

// JSON file implementation (simpler, no external deps)
pub mod json_store;
```

### SQLite Implementation

```rust
// fpl-storage/src/sqlite.rs

use rusqlite::{Connection, params};
use std::path::Path;

pub struct SqliteStorage {
    conn: Connection,
}

impl SqliteStorage {
    pub fn new(path: &Path) -> Result<Self, StorageError> {
        let conn = Connection::open(path)?;
        
        // Run migrations
        conn.execute_batch(include_str!("../migrations/001_initial.sql"))?;
        
        Ok(Self { conn })
    }
    
    pub fn in_memory() -> Result<Self, StorageError> {
        let conn = Connection::open_in_memory()?;
        conn.execute_batch(include_str!("../migrations/001_initial.sql"))?;
        Ok(Self { conn })
    }
}

#[async_trait]
impl Storage for SqliteStorage {
    async fn save_run(&self, run: &Run) -> Result<(), StorageError> {
        self.conn.execute(
            "INSERT INTO runs (id, created_at, config_hash, config_json, team_id, 
                              next_gameweek, horizon, status) 
             VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8)",
            params![
                run.id,
                run.created_at.to_rfc3339(),
                run.config_hash,
                serde_json::to_string(&run.config)?,
                run.team_id,
                run.next_gameweek,
                run.horizon,
                format!("{:?}", run.status).to_lowercase(),
            ],
        )?;
        Ok(())
    }
    
    async fn save_solution(&self, solution: &Solution, run_id: &str) -> Result<(), StorageError> {
        let tx = self.conn.transaction()?;
        
        // Insert solution
        tx.execute(
            "INSERT INTO solutions (id, run_id, iteration, score, total_expected_points, 
                                   total_hits, summary_json)
             VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7)",
            params![
                solution.id,
                run_id,
                solution.iteration,
                solution.score,
                solution.total_expected_points(),
                solution.total_hits(),
                serde_json::to_string(solution)?,
            ],
        )?;
        
        // Insert gameweek plans
        for gw in &solution.gameweeks {
            tx.execute(
                "INSERT INTO gameweek_plans (solution_id, gameweek, expected_points, bank,
                                            free_transfers, transfers_made, hits_taken,
                                            chip_used, picks_json)
                 VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9)",
                params![
                    solution.id,
                    gw.gameweek,
                    gw.expected_points,
                    gw.bank,
                    gw.free_transfers,
                    gw.transfers_made,
                    gw.hits_taken,
                    gw.chip_used.map(|c| c.short_name()),
                    serde_json::to_string(&gw.picks)?,
                ],
            )?;
        }
        
        tx.commit()?;
        Ok(())
    }
    
    async fn get_recent_runs(&self, limit: usize) -> Result<Vec<Run>, StorageError> {
        let mut stmt = self.conn.prepare(
            "SELECT id, created_at, config_hash, config_json, team_id, 
                    next_gameweek, horizon, status, error_message, solve_time_secs
             FROM runs
             ORDER BY created_at DESC
             LIMIT ?1"
        )?;
        
        let runs = stmt.query_map([limit], |row| {
            Ok(Run {
                id: row.get(0)?,
                created_at: chrono::DateTime::parse_from_rfc3339(&row.get::<_, String>(1)?)
                    .unwrap()
                    .with_timezone(&chrono::Utc),
                config_hash: row.get(2)?,
                config: serde_json::from_str(&row.get::<_, String>(3)?).unwrap(),
                team_id: row.get(4)?,
                next_gameweek: row.get(5)?,
                horizon: row.get(6)?,
                status: match row.get::<_, String>(7)?.as_str() {
                    "running" => RunStatus::Running,
                    "completed" => RunStatus::Completed,
                    _ => RunStatus::Failed,
                },
                error_message: row.get(8)?,
                solve_time_secs: row.get(9)?,
            })
        })?.collect::<Result<Vec<_>, _>>()?;
        
        Ok(runs)
    }
    
    // ... other implementations
}
```

### CLI History Commands

```rust
// fpl-cli/src/commands/history.rs

use clap::Subcommand;

#[derive(Subcommand)]
pub enum HistoryCommand {
    /// List recent optimization runs
    List {
        #[arg(short, long, default_value = "10")]
        limit: usize,
        
        #[arg(long)]
        team_id: Option<u32>,
    },
    
    /// Show details of a specific run
    Show {
        /// Run ID or "latest"
        run_id: String,
    },
    
    /// Compare two runs
    Compare {
        run_id_1: String,
        run_id_2: String,
    },
    
    /// Export run results
    Export {
        run_id: String,
        
        #[arg(short, long, default_value = "json")]
        format: ExportFormat,
        
        #[arg(short, long)]
        output: Option<PathBuf>,
    },
    
    /// Delete old runs
    Prune {
        #[arg(long, default_value = "50")]
        keep: usize,
    },
}

pub async fn handle_history(cmd: HistoryCommand, storage: &dyn Storage) -> Result<()> {
    match cmd {
        HistoryCommand::List { limit, team_id } => {
            let runs = if let Some(tid) = team_id {
                storage.search_runs(RunQuery { team_id: Some(tid), limit: Some(limit), ..Default::default() }).await?
            } else {
                storage.get_recent_runs(limit).await?
            };
            
            print_runs_table(&runs);
        }
        
        HistoryCommand::Show { run_id } => {
            let run = if run_id == "latest" {
                storage.get_recent_runs(1).await?.into_iter().next()
            } else {
                storage.get_run(&run_id).await?
            };
            
            if let Some(run) = run {
                print_run_details(&run);
                let solutions = storage.get_solutions(&run.id).await?;
                print_solutions(&solutions);
            } else {
                println!("Run not found");
            }
        }
        
        // ... other handlers
    }
    
    Ok(())
}
```

---

## 9. Docker Deployment

### Dockerfile

```dockerfile
# Build stage
FROM rust:1.75-slim-bookworm AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy manifests
COPY Cargo.toml Cargo.lock ./
COPY crates ./crates

# Build release binary
RUN cargo build --release --bin fpl

# Runtime stage
FROM debian:bookworm-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -m -u 1000 fpl
USER fpl

WORKDIR /home/fpl

# Copy binary from builder
COPY --from=builder /app/target/release/fpl /usr/local/bin/fpl

# Create directories
RUN mkdir -p /home/fpl/data /home/fpl/config /home/fpl/output

# Set default config path
ENV FPL_CONFIG_PATH=/home/fpl/config/config.json
ENV FPL_DATA_DIR=/home/fpl/data
ENV FPL_DATABASE_PATH=/home/fpl/output/history.db

# Volume mounts
VOLUME ["/home/fpl/data", "/home/fpl/config", "/home/fpl/output"]

ENTRYPOINT ["fpl"]
CMD ["--help"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  fpl-optimizer:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fpl-optimizer
    
    volumes:
      # Mount local data directory
      - ./data:/home/fpl/data:ro
      # Mount config
      - ./config:/home/fpl/config:ro
      # Persist results
      - fpl-output:/home/fpl/output
    
    environment:
      - FPL_TEAM_ID=${FPL_TEAM_ID:-}
      - FPL_VERBOSE=true
      - RUST_LOG=fpl=info
    
    # For one-off runs
    profiles:
      - cli
    
    command: ["solve", "--config", "/home/fpl/config/config.json"]

  # Optional: scheduled runs
  fpl-cron:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fpl-cron
    
    volumes:
      - ./data:/home/fpl/data:ro
      - ./config:/home/fpl/config:ro
      - fpl-output:/home/fpl/output
    
    environment:
      - FPL_TEAM_ID=${FPL_TEAM_ID}
    
    # Run daily at 6 AM
    entrypoint: >
      sh -c "while true; do
        fpl solve --config /home/fpl/config/config.json;
        sleep 86400;
      done"
    
    profiles:
      - scheduled

volumes:
  fpl-output:
```

### Usage Examples

```bash
# Build the image
docker build -t fpl-optimizer .

# Run a single optimization
docker run --rm \
  -v $(pwd)/data:/home/fpl/data:ro \
  -v $(pwd)/config:/home/fpl/config:ro \
  -v $(pwd)/output:/home/fpl/output \
  -e FPL_TEAM_ID=12345 \
  fpl-optimizer solve

# Run with docker-compose
FPL_TEAM_ID=12345 docker-compose run --rm fpl-optimizer solve

# Interactive shell
docker run --rm -it \
  -v $(pwd)/data:/home/fpl/data:ro \
  -v $(pwd)/config:/home/fpl/config:ro \
  fpl-optimizer history list

# View logs from previous runs
docker-compose logs fpl-optimizer
```

### Multi-arch Build

```bash
# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/username/fpl-optimizer:latest \
  --push .
```

---

## 10. Error Handling

### Error Types Hierarchy

```rust
// fpl-core/src/error.rs

use thiserror::Error;

/// Top-level application error
#[derive(Error, Debug)]
pub enum FplError {
    #[error("Configuration error: {0}")]
    Config(#[from] ConfigError),
    
    #[error("API error: {0}")]
    Api(#[from] ApiError),
    
    #[error("Data processing error: {0}")]
    Data(#[from] DataError),
    
    #[error("Solver error: {0}")]
    Solver(#[from] SolverError),
    
    #[error("Storage error: {0}")]
    Storage(#[from] StorageError),
    
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

#[derive(Error, Debug)]
pub enum ConfigError {
    #[error("Failed to read config file: {0}")]
    ReadError(#[source] std::io::Error),
    
    #[error("Failed to parse config: {0}")]
    ParseError(#[source] serde_json::Error),
    
    #[error("Invalid configuration: {message}")]
    ValidationError { message: String, field: String },
    
    #[error("Missing required field: {0}")]
    MissingField(String),
}

#[derive(Error, Debug)]
pub enum ApiError {
    #[error("Network error: {0}")]
    Network(#[source] reqwest::Error),
    
    #[error("HTTP error {status}: {message}")]
    HttpError { status: u16, message: String },
    
    #[error("Failed to parse API response: {0}")]
    ParseError(#[source] serde_json::Error),
    
    #[error("Rate limited, retry after {retry_after} seconds")]
    RateLimited { retry_after: u64 },
    
    #[error("Team not found: {0}")]
    TeamNotFound(u32),
}

#[derive(Error, Debug)]
pub enum DataError {
    #[error("Projection file not found: {0}")]
    FileNotFound(std::path::PathBuf),
    
    #[error("Invalid CSV format: {0}")]
    CsvError(String),
    
    #[error("Missing column: {0}")]
    MissingColumn(String),
    
    #[error("Player not found: {0}")]
    PlayerNotFound(u32),
    
    #[error("No projection data for gameweek {0}")]
    MissingGameweek(u8),
}

#[derive(Error, Debug)]
pub enum SolverError {
    #[error("Failed to build model: {0}")]
    BuildError(String),
    
    #[error("Solver returned infeasible")]
    Infeasible,
    
    #[error("Solver timed out after {0} seconds")]
    Timeout(u64),
    
    #[error("No solution found")]
    NoSolution,
    
    #[error("Solver internal error: {0}")]
    InternalError(String),
}

#[derive(Error, Debug)]
pub enum StorageError {
    #[error("Database error: {0}")]
    Database(#[source] rusqlite::Error),
    
    #[error("Serialization error: {0}")]
    Serialization(#[source] serde_json::Error),
    
    #[error("Run not found: {0}")]
    RunNotFound(String),
}
```

### Error Handling Patterns

```rust
// Using anyhow for CLI with context
use anyhow::{Context, Result};

async fn run_solve(config: Config) -> Result<()> {
    let api_client = FplClient::new();
    
    let bootstrap = api_client
        .get_bootstrap()
        .await
        .context("Failed to fetch FPL bootstrap data")?;
    
    let team_data = api_client
        .get_team(config.team_id.context("team_id is required")?)
        .await
        .context("Failed to fetch team data")?;
    
    let projections = load_projections(&config)
        .context("Failed to load projection data")?;
    
    let solution = solve(bootstrap, team_data, projections, &config)
        .context("Optimization failed")?;
    
    Ok(())
}

// Using Result<T, E> for libraries
impl FplModel {
    pub fn build(&mut self) -> Result<(), ModelError> {
        self.validate_inputs()?;
        
        self.create_variables()
            .map_err(|e| ModelError::BuildError(format!("Variable creation failed: {}", e)))?;
        
        self.add_constraints()
            .map_err(|e| ModelError::BuildError(format!("Constraint addition failed: {}", e)))?;
        
        Ok(())
    }
}
```

---

## 11. Testing Strategy

### Test Organization

```
tests/
├── integration/
│   ├── api_tests.rs        # API client integration tests
│   ├── solver_tests.rs     # End-to-end solver tests
│   └── storage_tests.rs    # Database tests
├── fixtures/
│   ├── bootstrap.json      # Sample API responses
│   ├── team.json
│   ├── projections.csv
│   └── expected_solution.json
└── common/
    └── mod.rs              # Shared test utilities
```

### Unit Tests

```rust
// fpl-core/src/player.rs

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_position_lineup_limits() {
        assert_eq!(Position::Goalkeeper.lineup_min(), 1);
        assert_eq!(Position::Goalkeeper.lineup_max(), 1);
        assert_eq!(Position::Defender.lineup_min(), 3);
        assert_eq!(Position::Defender.lineup_max(), 5);
    }
    
    #[test]
    fn test_selling_price_calculation() {
        // No profit
        assert_eq!(OwnedPlayer::calculate_selling_price(100, 100), 100);
        
        // Price increased by 2 (sell at +1)
        assert_eq!(OwnedPlayer::calculate_selling_price(100, 102), 101);
        
        // Price increased by 5 (sell at +2)
        assert_eq!(OwnedPlayer::calculate_selling_price(100, 105), 102);
        
        // Price decreased
        assert_eq!(OwnedPlayer::calculate_selling_price(100, 95), 95);
    }
    
    #[test]
    fn test_player_total_ev() {
        let player = Player {
            id: 1,
            name: "Test".into(),
            full_name: None,
            position: Position::Midfielder,
            team_id: 1,
            team_name: "Test FC".into(),
            price: 100,
            projections: [
                (22, Projection { expected_points: 5.0, expected_minutes: 90.0 }),
                (23, Projection { expected_points: 6.0, expected_minutes: 85.0 }),
            ].into_iter().collect(),
        };
        
        assert!((player.total_expected_points() - 11.0).abs() < 0.001);
        assert!((player.total_expected_minutes() - 175.0).abs() < 0.001);
    }
}
```

### Integration Tests

```rust
// tests/integration/solver_tests.rs

use fpl_core::{Config, Player, SquadState};
use fpl_solver::FplModel;

#[test]
fn test_basic_optimization() {
    // Load test fixtures
    let players: Vec<Player> = serde_json::from_str(
        include_str!("../fixtures/players.json")
    ).unwrap();
    
    let initial_state: SquadState = serde_json::from_str(
        include_str!("../fixtures/initial_squad.json")
    ).unwrap();
    
    let config = Config {
        horizon: 3,
        ..Default::default()
    };
    
    let mut model = FplModel::new(players, initial_state, config, 22);
    model.build().expect("Model should build");
    
    let solution = model.solve().expect("Should find solution");
    
    // Verify solution validity
    assert_eq!(solution.gameweeks.len(), 3);
    
    for gw in &solution.gameweeks {
        // Check squad size
        assert_eq!(gw.picks.iter().filter(|p| !p.is_transfer_out).count(), 15);
        
        // Check lineup size
        assert_eq!(gw.lineup().count(), 11);
        
        // Check captain exists
        assert!(gw.captain().is_some());
    }
}

#[tokio::test]
async fn test_api_integration() {
    // Use wiremock for mocking
    use wiremock::{MockServer, Mock, ResponseTemplate};
    use wiremock::matchers::{method, path};
    
    let mock_server = MockServer::start().await;
    
    Mock::given(method("GET"))
        .and(path("/api/bootstrap-static/"))
        .respond_with(ResponseTemplate::new(200)
            .set_body_json(include_str!("../fixtures/bootstrap.json")))
        .mount(&mock_server)
        .await;
    
    let client = FplClient::with_base_url(&mock_server.uri());
    let bootstrap = client.get_bootstrap().await.unwrap();
    
    assert!(!bootstrap.elements.is_empty());
}
```

### Property-Based Tests

```rust
// Using proptest for property-based testing
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_config_roundtrip(
        horizon in 1u8..20,
        decay in 0.5f64..1.0,
        team_id in proptest::option::of(1u32..10_000_000),
    ) {
        let config = Config {
            horizon,
            decay_base: decay,
            team_id,
            ..Default::default()
        };
        
        let json = serde_json::to_string(&config).unwrap();
        let parsed: Config = serde_json::from_str(&json).unwrap();
        
        assert_eq!(config.horizon, parsed.horizon);
        assert!((config.decay_base - parsed.decay_base).abs() < 0.0001);
        assert_eq!(config.team_id, parsed.team_id);
    }
}
```

---

## 12. Performance Considerations

### Memory Optimization

```rust
// Use Arc for shared read-only data
use std::sync::Arc;

pub struct OptimizationContext {
    pub players: Arc<Vec<Player>>,
    pub bootstrap: Arc<BootstrapData>,
}

// Use SmallVec for small collections
use smallvec::SmallVec;

pub struct GameweekPlan {
    // Most squads have exactly 15 players
    pub picks: SmallVec<[PlayerPick; 16]>,
}

// Use interned strings for repeated values
use string_cache::DefaultAtom as Atom;

pub struct Player {
    pub name: Atom,
    pub team_name: Atom,
}
```

### Parallelization

```rust
use rayon::prelude::*;

// Parallel filtering
let filtered_players: Vec<Player> = players
    .par_iter()
    .filter(|p| p.total_expected_minutes() >= config.xmin_lb)
    .cloned()
    .collect();

// Parallel solution iterations
let solutions: Vec<Solution> = (0..config.num_iterations)
    .into_par_iter()
    .map(|i| {
        let mut model = FplModel::new(/* ... */);
        model.add_iteration_cut(i);
        model.solve()
    })
    .collect::<Result<Vec<_>, _>>()?;
```

### Caching Strategy

```rust
// fpl-api/src/cache.rs

use std::collections::HashMap;
use std::time::{Duration, Instant};

pub struct Cache {
    entries: HashMap<String, CacheEntry>,
    ttl: Duration,
}

struct CacheEntry {
    data: String,
    inserted_at: Instant,
}

impl Cache {
    pub fn get(&self, key: &str) -> Option<&str> {
        self.entries.get(key).and_then(|entry| {
            if entry.inserted_at.elapsed() < self.ttl {
                Some(entry.data.as_str())
            } else {
                None
            }
        })
    }
    
    pub fn set(&mut self, key: String, data: String) {
        self.entries.insert(key, CacheEntry {
            data,
            inserted_at: Instant::now(),
        });
    }
    
    pub fn clear_expired(&mut self) {
        self.entries.retain(|_, entry| entry.inserted_at.elapsed() < self.ttl);
    }
}
```

### Benchmarks

```rust
// benches/solver_bench.rs

use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};

fn bench_model_build(c: &mut Criterion) {
    let players = load_test_players();
    let config = Config::default();
    
    c.bench_function("model_build_500_players", |b| {
        b.iter(|| {
            let mut model = FplModel::new(players.clone(), initial_state.clone(), config.clone(), 22);
            model.build()
        })
    });
}

fn bench_solver(c: &mut Criterion) {
    let mut group = c.benchmark_group("solver");
    group.sample_size(10);  // Fewer samples for slow benchmarks
    
    for horizon in [1, 3, 5, 8] {
        group.bench_with_input(
            BenchmarkId::new("horizon", horizon),
            &horizon,
            |b, &h| {
                let config = Config { horizon: h, ..Default::default() };
                let mut model = FplModel::new(/* ... */);
                model.build().unwrap();
                
                b.iter(|| model.solve())
            },
        );
    }
    
    group.finish();
}

criterion_group!(benches, bench_model_build, bench_solver);
criterion_main!(benches);
```

---

## Appendix A: CLI Command Reference

```
fpl - Fantasy Premier League Squad Optimizer

USAGE:
    fpl [OPTIONS] <COMMAND>

OPTIONS:
    -c, --config <PATH>      Configuration file path
    -v, --verbose            Enable verbose output
    --json                   Output in JSON format
    --log-file <PATH>        Write logs to file
    -h, --help               Print help
    -V, --version            Print version

COMMANDS:
    solve       Run squad optimization
    simulate    Run Monte Carlo simulations
    history     View and manage run history
    config      Manage configuration
    help        Print help for a command

EXAMPLES:
    # Basic solve with team ID
    fpl solve --team-id 12345
    
    # Solve with custom config
    fpl solve --config my_config.json
    
    # Run 100 simulations
    fpl simulate --runs 100 --parallel 4
    
    # View recent runs
    fpl history list --limit 5
    
    # Export latest solution
    fpl history export latest --format csv -o solution.csv
```

---

## Appendix B: Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `FPL_TEAM_ID` | Your FPL team ID | None |
| `FPL_CONFIG_PATH` | Path to config file | `./config.json` |
| `FPL_DATA_DIR` | Directory for data files | `./data` |
| `FPL_DATABASE_PATH` | SQLite database path | `./history.db` |
| `FPL_LOG_LEVEL` | Log level (trace/debug/info/warn/error) | `info` |
| `FPL_LOG_FORMAT` | Log format (pretty/json) | `pretty` |
| `RUST_LOG` | Standard Rust log filter | None |

---

*End of Rust Design Document*
