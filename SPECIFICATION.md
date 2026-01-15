# FPL Squad Optimizer - Technical Specification

> **Language-Agnostic Design Document**
> 
> This document specifies the complete functionality of an FPL (Fantasy Premier League) squad optimization system. It is designed to be implementation-independent and can serve as a blueprint for any programming language.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Data Structures](#2-data-structures)
3. [External APIs](#3-external-apis)
4. [Data Processing Pipeline](#4-data-processing-pipeline)
5. [Optimization Model](#5-optimization-model)
6. [Configuration System](#6-configuration-system)
7. [Output & Reporting](#7-output--reporting)
8. [Advanced Features](#8-advanced-features)
9. [CLI Interface](#9-cli-interface)
10. [File Formats](#10-file-formats)

---

## 1. System Overview

### 1.1 Purpose

The system solves the **multi-period Fantasy Premier League squad selection problem** using Mixed Integer Linear Programming (MILP). It determines:

- Which 15 players to have in the squad each gameweek
- Which 11 players to start (lineup)
- Who should be captain and vice-captain
- Bench order for auto-substitutions
- Which transfers to make and when
- When to use chips (Wildcard, Free Hit, Bench Boost, Triple Captain)

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INPUT LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ FPL API     │  │ Projection  │  │ Team Data   │  │ Settings    │        │
│  │ (bootstrap) │  │ CSV Files   │  │ (JSON)      │  │ (JSON)      │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PROCESSING LAYER                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Data Parser │  │ Data Merger │  │ Player      │  │ Team JSON   │        │
│  │             │  │             │  │ Filter      │  │ Generator   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          OPTIMIZATION LAYER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      MILP MODEL BUILDER                              │   │
│  │  • Decision Variables (squad, lineup, captain, transfers, chips)    │   │
│  │  • Constraints (squad size, budget, formation, team limits)         │   │
│  │  • Objective Function (expected points with decay)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         MIP SOLVER                                   │   │
│  │                    (HiGHS, Gurobi, CBC, etc.)                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            OUTPUT LAYER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ CSV Results │  │ Console     │  │ Timeline    │  │ Sensitivity │        │
│  │             │  │ Summary     │  │ Image       │  │ Analysis    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Core Modules

| Module | Responsibility |
|--------|----------------|
| **Data Parser** | Read projection data from various CSV formats |
| **Data Merger** | Combine FPL API data with projection data |
| **Team Generator** | Build team state from FPL API or JSON |
| **Model Builder** | Construct MILP optimization model |
| **Solver Interface** | Export to MPS, invoke solver, parse solution |
| **Result Processor** | Convert solution to human-readable output |
| **Visualization** | Generate squad timeline images |
| **Sensitivity Analyzer** | Aggregate results from multiple runs |

---

## 2. Data Structures

### 2.1 Player

```
Player {
    id: Integer              // FPL element ID (unique)
    name: String             // Display name (web_name)
    full_name: String        // First + Second name
    position: Enum           // G (1), D (2), M (3), F (4)
    team_id: Integer         // Team ID (1-20)
    team_name: String        // Team name
    price: Decimal           // Current price in £M (e.g., 13.5)
    
    // Per-gameweek projections
    projections: Map<Gameweek, PlayerProjection>
}

PlayerProjection {
    expected_points: Decimal  // xPts for this gameweek
    expected_minutes: Decimal // xMins for this gameweek
}
```

### 2.2 Squad State

```
SquadState {
    players: List<SquadPlayer>     // Exactly 15 players
    bank: Decimal                  // Money in the bank (£M)
    free_transfers: Integer        // Available free transfers (1-5)
    chips_available: ChipStatus
    chips_used: List<ChipUsage>
}

SquadPlayer {
    player_id: Integer
    purchase_price: Decimal        // Price when bought
    selling_price: Decimal         // Current sell value (may differ)
}

ChipStatus {
    wildcard_available: Boolean
    free_hit_available: Boolean
    bench_boost_available: Boolean
    triple_captain_available: Boolean
}

ChipUsage {
    chip: Enum                     // WC, FH, BB, TC
    gameweek: Integer
}
```

### 2.3 Solution

```
Solution {
    score: Decimal                 // Objective function value
    gameweeks: List<GameweekPlan>
    summary: String
    statistics: Map<Gameweek, GameweekStats>
}

GameweekPlan {
    gameweek: Integer
    squad: List<PlayerPick>        // 15 players
    lineup: List<PlayerPick>       // 11 starters
    bench: List<PlayerPick>        // 4 bench (ordered)
    captain: PlayerPick
    vice_captain: PlayerPick
    transfers_in: List<Player>
    transfers_out: List<Player>
    chip_used: Optional<Chip>
    hits_taken: Integer
}

PlayerPick {
    player: Player
    expected_points: Decimal
    expected_minutes: Decimal
    is_captain: Boolean
    is_vice_captain: Boolean
    multiplier: Integer            // 1, 2 (captain), or 3 (TC)
}

GameweekStats {
    expected_points: Decimal
    bank: Decimal
    free_transfers: Integer
    transfers_made: Integer
    hits_taken: Integer
    chip: Optional<Chip>
}
```

### 2.4 Configuration

```
Config {
    // Planning
    horizon: Integer               // Gameweeks to plan ahead (default: 8)
    
    // Objective tuning
    decay_base: Decimal            // Future gameweek discount (default: 0.9)
    bench_weights: Map<BenchPosition, Decimal>  // Bench point weights
    vcap_weight: Decimal           // Vice-captain weight (default: 0.1)
    ft_value: Decimal              // Value of rolling a free transfer
    ft_value_list: Map<FTState, Decimal>  // Per-state FT values
    itb_value: Decimal             // Value of £1M in bank
    hit_cost: Integer              // Points per hit (default: 4)
    
    // Data source
    datasource: String             // CSV filename or "mixed"
    data_weights: Map<String, Decimal>  // Weights for mixed data
    
    // Team data
    team_data: Enum                // "id", "json", or file
    team_id: Optional<Integer>     // FPL team ID
    
    // Player constraints
    banned: List<Integer>          // Player IDs to exclude
    locked: List<Integer>          // Player IDs to always include
    
    // Transfer constraints
    no_transfer_last_gws: Integer  // GWs at end with no transfers
    hit_limit: Optional<Integer>   // Max total hits
    weekly_hit_limit: Optional<Integer>  // Max hits per GW
    
    // Chip scheduling
    use_wc: List<Integer>          // GWs to use Wildcard
    use_bb: List<Integer>          // GWs to use Bench Boost
    use_fh: List<Integer>          // GWs to use Free Hit
    use_tc: List<Integer>          // GWs to use Triple Captain
    chip_limits: Map<Chip, Integer>
    
    // Filtering
    xmin_lb: Decimal               // Min expected minutes threshold
    ev_per_price_cutoff: Decimal   // Percentile cutoff for EV/price
    keep_top_ev_percent: Decimal   // Always keep top N% by EV
    
    // Solver settings
    solver: String                 // "highs", "gurobi", "cbc"
    time_limit_secs: Integer       // Max solve time
    optimality_gap: Decimal        // Acceptable gap from optimal
    
    // Output
    num_iterations: Integer        // Alternative solutions to generate
    print_squads: Boolean
    export_image: Boolean
}
```

---

## 3. External APIs

### 3.1 FPL Bootstrap API

**Endpoint**: `https://fantasy.premierleague.com/api/bootstrap-static/`

**Purpose**: Get all player data, team data, and gameweek information.

**Response Structure** (relevant fields):

```
{
    "elements": [                    // All players
        {
            "id": 1,
            "web_name": "Salah",
            "first_name": "Mohamed",
            "second_name": "Salah",
            "team": 11,              // Team ID
            "team_code": 14,         // Team code
            "element_type": 3,       // Position (1=GK, 2=DEF, 3=MID, 4=FWD)
            "now_cost": 130,         // Price in £0.1M
            "cost_change_start": 5,  // Price change since season start
            ...
        }
    ],
    "teams": [
        {
            "id": 1,
            "name": "Arsenal",
            "short_name": "ARS",
            "code": 3
        }
    ],
    "events": [                      // Gameweeks
        {
            "id": 1,
            "is_current": false,
            "is_next": true,
            "deadline_time": "2024-08-16T17:30:00Z"
        }
    ],
    "element_types": [               // Positions
        {
            "id": 1,
            "singular_name": "Goalkeeper",
            "singular_name_short": "GKP",
            "squad_select": 2,       // Required in squad
            "squad_min_play": 1,     // Min in lineup
            "squad_max_play": 1      // Max in lineup
        }
    ]
}
```

### 3.2 FPL Team API

**Endpoint**: `https://fantasy.premierleague.com/api/my-team/{team_id}/`

**Purpose**: Get current squad, chips, and transfer info.

**Response Structure**:

```
{
    "picks": [
        {
            "element": 307,          // Player ID
            "position": 1,           // Squad position (1-15)
            "selling_price": 55,     // Sell price in £0.1M
            "purchase_price": 55,    // Buy price in £0.1M
            "multiplier": 1,         // 0=bench, 1=play, 2=captain
            "is_captain": false,
            "is_vice_captain": false
        }
    ],
    "chips": [
        {
            "name": "wildcard",
            "status_for_entry": "available"  // or "played"
        }
    ],
    "transfers": {
        "bank": 15,                  // Bank in £0.1M
        "limit": 2,                  // Free transfers available
        "made": 0,                   // Transfers made this GW
        "cost": 4                    // Cost per extra transfer
    }
}
```

### 3.3 FPL Fixtures API

**Endpoint**: `https://fantasy.premierleague.com/api/fixtures/`

**Purpose**: Get fixture schedule for opposing-play constraints.

### 3.4 FPL Transfer History API

**Endpoint**: `https://fantasy.premierleague.com/api/entry/{team_id}/transfers/`

**Purpose**: Calculate current free transfer count from history.

---

## 4. Data Processing Pipeline

### 4.1 Projection Data Formats

The system supports multiple projection data formats:

#### Format A: Standard (Solio/FPLReview)

```csv
ID,Name,Pos,Team,Value,22_Pts,22_xMins,23_Pts,23_xMins,...
301,Salah,M,Liverpool,13.0,7.5,90,6.8,85,...
427,Haaland,F,Man City,15.0,8.2,90,7.9,88,...
```

**Required columns**:
- `ID`: FPL player ID
- `Pos`: Position (G/D/M/F)
- `{GW}_Pts`: Expected points for gameweek
- `{GW}_xMins`: Expected minutes for gameweek

#### Format B: Mikkel Format

Different column structure requiring conversion. Uses BCV (Best Case Value) and weighted minutes.

### 4.2 Data Merging Algorithm

```
FUNCTION merge_data(fpl_data, projection_data):
    merged = empty_dataframe
    
    FOR each player IN fpl_data.elements:
        projection = find_by_id(projection_data, player.id)
        
        IF projection EXISTS:
            merged.add(combine(player, projection))
        ELSE:
            # Add player with zero projections
            merged.add(player_with_zero_projections(player))
    
    RETURN merged
```

### 4.3 Player Filtering

Players are filtered to reduce problem size:

```
FUNCTION filter_players(players, config, current_squad):
    safe_players = current_squad 
                 + config.locked 
                 + config.keep 
                 + top_ev_players(config.keep_top_ev_percent)
    
    filtered = players.filter(
        p => p.total_xmins >= config.xmin_lb
          OR p.id IN safe_players
    )
    
    IF config.ev_per_price_cutoff > 0:
        cutoff = percentile(filtered, config.ev_per_price_cutoff, by=ev/price)
        filtered = filtered.filter(
            p => p.ev/p.price > cutoff
              OR p.id IN safe_players
        )
    
    RETURN filtered
```

### 4.4 Free Transfer Calculation

When using team_id mode, calculate FTs from transfer history:

```
FUNCTION calculate_fts(transfers, first_gw, current_gw, fh_gws, wc_gws):
    n_transfers = count_transfers_per_gw(transfers)
    fts = {}
    fts[first_gw + 1] = 1
    
    FOR gw FROM first_gw + 2 TO current_gw + 1:
        IF gw - 1 IN fh_gws OR gw - 1 IN wc_gws:
            fts[gw] = fts[gw - 1]  # FTs preserved after chips
            CONTINUE
        
        # Normal FT calculation
        fts[gw] = fts[gw - 1] - n_transfers[gw - 1] + 1
        fts[gw] = clamp(fts[gw], 1, 5)
    
    RETURN fts[current_gw]
```

---

## 5. Optimization Model

### 5.1 Sets

```
P = set of all players (after filtering)
T = set of player types/positions {1, 2, 3, 4}
K = set of teams {1, 2, ..., 20}
W = set of gameweeks in horizon {w₀, w₀+1, ..., w₀+H-1}
W_all = {w₀-1} ∪ W  (includes previous GW for initial state)
O = bench order positions {0, 1, 2, 3}
S = free transfer states {0, 1, 2, 3, 4, 5}
```

### 5.2 Parameters

```
// Player data
xPts[p,w] = expected points for player p in gameweek w
xMin[p,w] = expected minutes for player p in gameweek w
type[p] = position type of player p
team[p] = team of player p
buy_price[p] = current purchase price of player p
sell_price[p] = selling price of player p (for owned players)

// Position limits
min_play[t] = minimum players of type t in lineup
max_play[t] = maximum players of type t in lineup
squad_req[t] = required players of type t in squad

// Initial state
initial_squad[p] = 1 if player p is in initial squad, 0 otherwise
initial_itb = initial money in bank
initial_ft = initial free transfers

// Configuration
decay = decay base for future gameweeks
bench_weight[o] = weight for bench position o
vcap_weight = vice-captain objective weight
ft_value[s] = value of being in FT state s
itb_value = value per £1M in bank
hit_cost = points deducted per hit
```

### 5.3 Decision Variables

```
// Squad composition
squad[p,w] ∈ {0,1}      // Is player p in squad in gameweek w?
squad_fh[p,w] ∈ {0,1}   // Is player p in Free Hit squad in gameweek w?

// Lineup decisions
lineup[p,w] ∈ {0,1}     // Is player p in starting lineup in gameweek w?
captain[p,w] ∈ {0,1}    // Is player p captain in gameweek w?
vicecap[p,w] ∈ {0,1}    // Is player p vice-captain in gameweek w?
bench[p,w,o] ∈ {0,1}    // Is player p in bench position o in gameweek w?

// Transfers
transfer_in[p,w] ∈ {0,1}   // Transfer player p in during gameweek w?
transfer_out[p,w] ∈ {0,1}  // Transfer player p out during gameweek w?

// State tracking
itb[w] ≥ 0              // Money in the bank after gameweek w
fts[w] ∈ {0,1,...,5}    // Free transfers available in gameweek w
penalized_transfers[w] ≥ 0  // Number of hits taken in gameweek w

// Chips
use_wc[w] ∈ {0,1}       // Use Wildcard in gameweek w?
use_bb[w] ∈ {0,1}       // Use Bench Boost in gameweek w?
use_fh[w] ∈ {0,1}       // Use Free Hit in gameweek w?
use_tc[p,w] ∈ {0,1}     // Use Triple Captain on player p in gameweek w?
```

### 5.4 Constraints

#### Squad Structure
```
// Exactly 15 players in squad
∑(p∈P) squad[p,w] = 15                              ∀w ∈ W

// Exactly 11 in lineup (15 if Bench Boost)
∑(p∈P) lineup[p,w] = 11 + 4·use_bb[w]              ∀w ∈ W

// Exactly one captain
∑(p∈P) captain[p,w] = 1                             ∀w ∈ W

// Exactly one vice-captain
∑(p∈P) vicecap[p,w] = 1                             ∀w ∈ W

// One player per bench position (unless Bench Boost)
∑(p∈P) bench[p,w,o] = 1 - use_bb[w]                ∀w ∈ W, o ∈ {1,2,3}

// Bench position 0 must be goalkeeper
∑(p∈P: type[p]=1) bench[p,w,0] = 1 - use_bb[w]     ∀w ∈ W
```

#### Lineup-Squad Relationships
```
// Can only play if in squad (or Free Hit squad)
lineup[p,w] ≤ squad[p,w] + use_fh[w]               ∀p ∈ P, w ∈ W
lineup[p,w] ≤ squad_fh[p,w] + 1 - use_fh[w]        ∀p ∈ P, w ∈ W

// Captain/VC must be in lineup
captain[p,w] ≤ lineup[p,w]                          ∀p ∈ P, w ∈ W
vicecap[p,w] ≤ lineup[p,w]                          ∀p ∈ P, w ∈ W

// Can't be both captain and vice-captain
captain[p,w] + vicecap[p,w] ≤ 1                     ∀p ∈ P, w ∈ W

// Can't be both in lineup and on bench
lineup[p,w] + ∑(o∈O) bench[p,w,o] ≤ 1              ∀p ∈ P, w ∈ W
```

#### Formation Constraints
```
// Valid formation (position minimums and maximums)
∑(p: type[p]=t) lineup[p,w] ≥ min_play[t]          ∀t ∈ T, w ∈ W
∑(p: type[p]=t) lineup[p,w] ≤ max_play[t] + use_bb[w]  ∀t ∈ T, w ∈ W

// Valid squad composition
∑(p: type[p]=t) squad[p,w] = squad_req[t]          ∀t ∈ T, w ∈ W
```

#### Team Limits
```
// Max 3 players from any team
∑(p: team[p]=k) squad[p,w] ≤ 3                     ∀k ∈ K, w ∈ W_all
∑(p: team[p]=k) squad_fh[p,w] ≤ 3·use_fh[w]        ∀k ∈ K, w ∈ W
```

#### Transfer Constraints
```
// Squad evolution through transfers
squad[p,w] = squad[p,w-1] + transfer_in[p,w] - transfer_out[p,w]
                                                    ∀p ∈ P, w ∈ W

// Budget constraint
itb[w] = itb[w-1] + ∑(p) sell_price[p]·transfer_out[p,w]
                  - ∑(p) buy_price[p]·transfer_in[p,w]
                                                    ∀w ∈ W

// Can't transfer in and out same player same week
transfer_in[p,w] + transfer_out[p,w] ≤ 1           ∀p ∈ P, w ∈ W

// No transfers during Free Hit
transfer_in[p,w] ≤ 1 - use_fh[w]                   ∀p ∈ P, w ∈ W
transfer_out[p,w] ≤ 1 - use_fh[w]                  ∀p ∈ P, w ∈ W
```

#### Free Transfer Logic
```
// Number of transfers made
num_transfers[w] = ∑(p) transfer_out[p,w]          ∀w ∈ W

// Penalized transfers (hits)
penalized_transfers[w] ≥ num_transfers[w] - fts[w] - 15·use_wc[w]
                                                    ∀w ∈ W

// FT state transitions (using big-M formulation)
// If used fewer than FTs: fts[w+1] = fts[w] - used + 1, capped at 5
// If used more than FTs: fts[w+1] = 1
// If used WC or FH: fts preserved
```

#### Chip Constraints
```
// Only one chip per gameweek
use_wc[w] + use_fh[w] + use_bb[w] + ∑(p) use_tc[p,w] ≤ 1    ∀w ∈ W

// TC only on captain
use_tc[p,w] ≤ captain[p,w]                         ∀p ∈ P, w ∈ W

// Chip limits over horizon
∑(w) use_wc[w] ≤ wc_limit
∑(w) use_fh[w] ≤ fh_limit
∑(w) use_bb[w] ≤ bb_limit
∑(w) ∑(p) use_tc[p,w] ≤ tc_limit
```

#### Initial Conditions
```
squad[p, w₀-1] = initial_squad[p]                  ∀p ∈ P
itb[w₀-1] = initial_itb
fts[w₀] = initial_ft · (1 - use_wc[w₀]) + use_wc[w₀]
```

### 5.5 Objective Function

```
// Points contribution per gameweek
gw_xp[w] = ∑(p∈P) xPts[p,w] · (
    lineup[p,w]                           // 1x for playing
    + captain[p,w]                        // +1x for captain
    + vcap_weight · vicecap[p,w]          // small weight for VC (tiebreaker)
    + use_tc[p,w]                         // +1x for triple captain
    + ∑(o∈O) bench_weight[o] · bench[p,w,o]  // small weights for bench
)

// FT value gain
ft_gain[w] = ft_value_function(fts[w]) - ft_value_function(fts[w-1])

// Total value per gameweek
gw_total[w] = gw_xp[w] 
            - hit_cost · penalized_transfers[w]
            + ft_gain[w]
            + itb_value · itb[w]

// Decayed objective (discount future gameweeks)
MAXIMIZE: ∑(w∈W) gw_total[w] · decay^(w - w₀)
```

---

## 6. Configuration System

### 6.1 Configuration Hierarchy

Settings are loaded in priority order (later overrides earlier):

```
1. comprehensive_settings.json  (base defaults)
2. user_settings.json           (user customization)
3. --config file1.json;file2.json  (additional config files)
4. Command-line arguments       (highest priority)
```

### 6.2 Key Settings Categories

#### Planning Settings
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `horizon` | int | 8 | Gameweeks to optimize |
| `decay_base` | float | 0.9 | Discount rate per gameweek |

#### Objective Weights
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `ft_value` | float | 1.5 | Base value of free transfer |
| `ft_value_list` | map | {2:2, 3:1.6, 4:1.3, 5:1.1} | Per-state FT values |
| `bench_weights` | map | {0:0.03, 1:0.21, 2:0.06, 3:0.002} | Bench position weights |
| `vcap_weight` | float | 0.1 | Vice-captain weight |
| `itb_value` | float | 0.08 | Value per £1M in bank |
| `hit_cost` | int | 4 | Points per transfer hit |

#### Player Filtering
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `xmin_lb` | int | 300 | Min expected minutes |
| `ev_per_price_cutoff` | int | 30 | EV/price percentile cutoff |
| `keep_top_ev_percent` | int | 5 | Always keep top N% |

#### Solver Settings
| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `solver` | string | "highs" | Solver to use |
| `secs` | int | 600 | Time limit in seconds |
| `gap` | float | 0 | Optimality gap tolerance |

---

## 7. Output & Reporting

### 7.1 Solution CSV Format

Each solution is saved as a CSV with columns:

```csv
id,week,name,pos,type,team,buy_price,sell_price,xP,xMin,squad,lineup,bench,captain,vicecaptain,transfer_in,transfer_out,multiplier,xp_cont,chip,iter,ft,transfer_count
301,22,Salah,MID,3,Liverpool,0,0,7.5,90,1,1,-1,0,0,0,0,1,7.5,,0,2,0
427,22,Haaland,FWD,4,Man City,0,0,8.2,90,1,1,-1,1,0,0,0,2,16.4,,0,2,0
```

### 7.2 Console Summary Format

```
** GW 22:
ITB=1.5->1.5, FT=2, PT=0, NT=0

Lineup: 
    Raya (5.2)
    Gabriel (6.1), Saliba (5.8), Robinson (4.2)
    Salah (7.5, C), Palmer (6.8), Saka (5.4), Rogers (4.1)
    Haaland (8.2), Watkins (5.1), Cunha (4.9)
Bench: 
    Flekken (2.1), Lewis (2.8), Wharton (2.4), Delap (3.1)
Lineup xPts: 67.52
```

### 7.3 Timeline Visualization

Generate a PNG image showing:
- Squad composition across gameweeks
- Transfer lines (bezier curves) connecting sold/bought players
- Captain/vice-captain indicators
- Chip usage highlights
- Expected points per gameweek

---

## 8. Advanced Features

### 8.1 Multiple Iterations

Generate alternative solutions by adding cuts:

```
AFTER finding solution i:
    
    IF iteration_criteria == "this_gw_transfer_in":
        // Force at least one different transfer in
        ADD CONSTRAINT: ∑(p: transfer_in[p,w₀]=1 in solution i) (1 - transfer_in[p,w₀])
                      + ∑(p: transfer_in[p,w₀]=0 in solution i) transfer_in[p,w₀] ≥ 1
    
    SOLVE again for solution i+1
```

### 8.2 Sensitivity Analysis (Simulations)

Run many solves with randomized projections to find robust transfers:

```
FOR run IN 1..N:
    // Add noise to projections
    FOR each player p, gameweek w:
        noise = xPts[p,w] · (92 - xMin[p,w]) / 134 · random_normal()
        xPts_noisy[p,w] = xPts[p,w] + noise · strength
    
    // Solve with noisy data
    solution[run] = solve(xPts_noisy)

// Aggregate results
transfer_counts = count(solution.transfers for all runs)
PRINT "Most common transfers:"
```

### 8.3 Binary Fixtures

Handle fixture uncertainty with multiple scenarios:

```
// Generate projection variants for different fixture outcomes
binary1.csv: Team A's GW33 fixture moves to GW34
binary2.csv: Team A's GW33 fixture moves to GW37

// Run weighted simulations
60% of runs use binary1.csv
40% of runs use binary2.csv
```

### 8.4 Booked Transfers

Pre-schedule specific transfers:

```json
{
    "booked_transfers": [
        {"gw": 25, "transfer_in": 427, "transfer_out": 301}
    ]
}
```

Adds constraint: `transfer_in[427, 25] = 1 AND transfer_out[301, 25] = 1`

### 8.5 Opposing Play Constraint

Prevent/penalize playing players who face each other:

```
IF no_opposing_play == true:
    // Hard constraint
    FOR each fixture (home_team vs away_team) in gameweek w:
        FOR p1 in home_team, p2 in away_team:
            lineup[p1, w] + lineup[p2, w] ≤ 1

ELIF no_opposing_play == "penalty":
    // Soft constraint with penalty
    ADD penalty variables and subtract from objective
```

---

## 9. CLI Interface

### 9.1 Main Entry Point

```bash
# Basic solve
fpl-solver solve

# With configuration
fpl-solver solve --config my_settings.json --horizon 5

# Override specific settings
fpl-solver solve --team_id 12345 --use_wc "[25]" --horizon 10
```

### 9.2 Commands

| Command | Description |
|---------|-------------|
| `solve` | Run optimization with current settings |
| `simulations` | Run multiple randomized solves |
| `sensitivity` | Analyze results from simulations |
| `parallel` | Run chip combination search |

### 9.3 Key Arguments

| Argument | Type | Description |
|----------|------|-------------|
| `--config` | string | Semicolon-separated config file paths |
| `--horizon` | int | Override planning horizon |
| `--team_id` | int | FPL team ID |
| `--datasource` | string | Projection file name |
| `--use_wc` | list | Gameweeks for Wildcard |
| `--use_bb` | list | Gameweeks for Bench Boost |
| `--use_fh` | list | Gameweeks for Free Hit |
| `--use_tc` | list | Gameweeks for Triple Captain |
| `--banned` | list | Player IDs to exclude |
| `--locked` | list | Player IDs to include |
| `--verbose` | bool | Print solver progress |

---

## 10. File Formats

### 10.1 MPS Format

The optimization model is exported to MPS format for solver consumption. See [MPS Format Specification](https://en.wikipedia.org/wiki/MPS_(format)).

Key sections:
- `ROWS`: Objective and constraints (N/E/L/G types)
- `COLUMNS`: Variable coefficients (column-wise storage)
- `RHS`: Right-hand side values
- `BOUNDS`: Variable bounds (BV for binary)

### 10.2 Team JSON Format

```json
{
    "picks": [
        {
            "element": 301,
            "position": 1,
            "selling_price": 130,
            "purchase_price": 125,
            "multiplier": 2,
            "is_captain": true,
            "is_vice_captain": false
        }
    ],
    "chips": [
        {"name": "wildcard", "status_for_entry": "available"}
    ],
    "transfers": {
        "bank": 15,
        "limit": 2,
        "made": 0
    }
}
```

### 10.3 Projection CSV Format

```csv
ID,Name,Pos,Team,Value,22_Pts,22_xMins,23_Pts,23_xMins,24_Pts,24_xMins
301,Salah,M,Liverpool,13.0,7.52,89.5,6.84,87.2,7.11,88.0
427,Haaland,F,Man City,15.0,8.21,90.0,7.95,89.1,8.05,90.0
```

---

## Appendix A: FPL Game Rules Reference

### Squad Composition
- 15 players total
- 2 Goalkeepers, 5 Defenders, 5 Midfielders, 3 Forwards
- Maximum 3 players from any single team
- Budget: £100M at season start

### Lineup Rules
- 11 starters from the 15
- Valid formations: 1 GK, 3-5 DEF, 2-5 MID, 1-3 FWD
- Captain earns 2x points
- Auto-substitution from bench if starter doesn't play

### Transfers
- 1 free transfer per gameweek
- Unused FT rolls over (max 5)
- Extra transfers cost 4 points each
- Selling price = purchase price + floor((current - purchase) / 2)

### Chips
- **Wildcard**: Unlimited free transfers for one gameweek
- **Free Hit**: Temporary unlimited transfers (reverts next GW)
- **Bench Boost**: All 15 players score points
- **Triple Captain**: Captain earns 3x points

---

## Appendix B: Solver Integration

### MPS Export Process

1. Build symbolic model with modeling library
2. Export to MPS text file
3. Solver reads MPS file
4. Solver returns solution vector
5. Map solution indices back to variable names

### Recommended Solvers

| Solver | License | MIP Support | Performance |
|--------|---------|-------------|-------------|
| HiGHS | MIT | Yes | Excellent |
| CBC | EPL | Yes | Good |
| GLPK | GPL | Yes | Moderate |
| Gurobi | Commercial | Yes | Best |
| CPLEX | Commercial | Yes | Best |

---

*End of Specification*
