# IPL Auction Simulator & Cricket Manager - Implementation Plan

This plan outlines the architecture, database schema, API logic, and frontend components to build the IPL Auction Simulator platform specified in [ipl_rules.md](file:///c:/Users/MP/Desktop/code/iplAuction/ipl_rules.md). The mathematical models for AI bidding and season simulation have been strictly defined.

## User Review Required

> [!IMPORTANT]
> Based on your request, we are bypassing local Docker/CLI development environments. The database and edge functions will be deployed and tested **directly against your live Supabase Cloud project**.
> 
> You will need to provide your Supabase Project URL and API Key so the frontend can connect, and we will execute the SQL schemas directly on your Cloud instance.
> 
> The dataset in [compiled_ipl_players_updated.csv](file:///C:/Users/MP/Downloads/compiled_ipl_players_updated.csv) will be used to automatically seed the `players` table on your cloud database.

## Proposed Changes

We will organize the project in a modular folder structure within the workspace:
* `C:/Users/MP/Desktop/code/iplAuction/supabase/` - SQL scripts and Edge Functions for deployment to the cloud.
* `C:/Users/MP/Desktop/code/iplAuction/frontend/` - React SPA (Vite + Zustand).

### 1. Database & Migrations Component (Supabase Cloud)

We will write SQL scripts to be executed directly in your Supabase Cloud SQL Editor.

#### [NEW] [schema.sql](file:///c:/Users/MP/Desktop/code/iplAuction/supabase/migrations/schema.sql)
Creates tables:
1. `stadium`: Columns for venues, spin assistance, pace bounce, boundary size.
2. `session`: Columns for unique multi-player session rooms and status.
3. `teams`: Columns for purse tracking, human flags, and denormalized optimization counters (`squad_size`, `overseas_count`).
4. `players`: Contains stats and simulation vectors.
5. `live_auction_state`: Handles temporary bid state, current top bidder, active countdown (15s).
6. `roster_slots`: Tracks player contracts signed.
7. `result`: Post-season simulation standings table.

#### [NEW] [seed.sql](file:///c:/Users/MP/Desktop/code/iplAuction/supabase/migrations/seed.sql)
Automatically converts the parsed player rows from [compiled_ipl_players_updated.csv](file:///C:/Users/MP/Downloads/compiled_ipl_players_updated.csv) into SQL insertion scripts, along with default IPL stadiums. These will be run on the Cloud database.

---

### 2. Backend Compute Component (Supabase Cloud Edge Functions + Hono)

Create a unified backend application hosting game loops, AI bidding engines, and the deterministic season simulation. These will be deployed directly to Supabase Cloud.

#### [NEW] [index.ts](file:///c:/Users/MP/Desktop/code/iplAuction/supabase/functions/auction-engine/index.ts)
A Deno-based Hono router exposing HTTP endpoints:
* `/session/create`, `/session/join`, `/session/lock`
* `/bid/place`: Validates bids using the **Rock-Bottom Uncapped Floor Boundary**.
* `/simulation/run`: Performs the 14-game round-robin offline math matching engine using the **Phase-Aware Evaluation Matrix**:
  * **Phase A (Powerplay 1-6):** `Batting Weight = (skill_vs_pace * 0.6) + (powerplay_aggression * 0.4 * pace_bounce)`. `Result Adjustment = Batting Weight * (1.5 - boundary_size)`
  * **Phase B (Middle 7-15):** `Batting Weight = skill_vs_spin * (1.5 - spin_assistance)`. Applies a 35% dismissal penalty for aggressive batters with low spin skill on spinning tracks.
  * **Phase C (Death 16-20):** `Phase Balance = (death_overs_execution * clutch_coefficient) - (opponent_bowler.death_overs_execution * boundary_size)`

#### [NEW] [ai-bidding-agent.ts](file:///c:/Users/MP/Desktop/code/iplAuction/supabase/functions/auction-engine/ai-bidding-agent.ts)
Deno script applying precise deterministic formulas to calculate AI Player Valuation Maximum ($V_{max} = base\_price \times (1.0 + Strategy\_Modifier \times Synergy\_Weight)$):
* `MONEYBALL`: Modifier based on average spin/pace skill. Hard cap at Base Price + 30%. Adds +0.25 synergy for uncapped players.
* `AGGRESSIVE`: Modifier based on max powerplay/death stats * clutch * 1.5. Cap at Base Price + 150%. Bids until the absolute uncapped floor boundary.
* `BALANCED`: Modifier averages pace/spin/clutch. Cap between Base Price + 40%-75%. Synergy drops to 0.0 if positional requirements are met.

---

### 3. Frontend Component (React + Vite + Zustand)

Build a high-performance single page application for multiplayer game coordination.

#### [NEW] [store.js](file:///c:/Users/MP/Desktop/code/iplAuction/frontend/src/store/store.js)
Zustand global store to manage real-time auction state, user team details, active bid values, and simulation outcomes. Configured to connect directly to the Supabase Cloud project URL.

#### [NEW] [FranchiseWarRoom.jsx](file:///c:/Users/MP/Desktop/code/iplAuction/frontend/src/pages/FranchiseWarRoom.jsx)
Page 1: Lobby room registry and venue pitch profile visualization.

#### [NEW] [ScoutingHub.jsx](file:///c:/Users/MP/Desktop/code/iplAuction/frontend/src/pages/ScoutingHub.jsx)
Page 2: Search filters and radar charts of player stats.

#### [NEW] [AuctionArena.jsx](file:///c:/Users/MP/Desktop/code/iplAuction/frontend/src/pages/AuctionArena.jsx)
Page 3: Live bidding hammer. Heavy visual elements, countdown ticking circle, budget validation alerts, and the "Panic Module".

#### [NEW] [SquadBlueprint.jsx](file:///c:/Users/MP/Desktop/code/iplAuction/frontend/src/pages/SquadBlueprint.jsx)
Page 4: 25-card slot grid displaying openers, finishers, keepers, and warning counters.

#### [NEW] [StandingsPanel.jsx](file:///c:/Users/MP/Desktop/code/iplAuction/frontend/src/pages/StandingsPanel.jsx)
Page 5: Post-season standings, matchups diagnostics, and value-asset franchise autopsy reports.

---

## Verification Plan

### Automated Tests
1. **API Endpoints Validation**:
   Run endpoint integration test checks using `curl` against the live Supabase Edge Function URLs.
2. **Mathematical Formulas**:
   Execute offline unit tests for the deterministic formulas (Floor algorithm, Evaluation Matrix) before deploying functions to the cloud.

### Manual Verification
1. Open multiple browser instances connecting to the live Supabase backend to simulate multi-user lobby lockout and real-time bidding synchronization.
2. Complete a live mock draft, verify that the cloud database updates ledger tables and purses correctly, and trigger the offline season simulation to verify standings logic.
