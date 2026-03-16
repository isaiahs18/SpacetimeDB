# Fantasy Football App Integration with SpacetimeDB

This guide walks you through building a real-time fantasy football backend using SpacetimeDB. By the end, your app will have live scoring updates, draft rooms, trade processing, and roster management — all handled inside the database with zero separate server infrastructure.

---

## Prerequisites

- [SpacetimeDB CLI installed](https://spacetimedb.com/install)
- Rust **or** C# **or** TypeScript for writing your module
- A client framework (React, Vue, Unity, etc.) for the front end
- Basic familiarity with relational database concepts

---

## 1. Install SpacetimeDB

```bash
# macOS / Linux
curl -sSf https://install.spacetimedb.com | sh

# Windows (PowerShell)
iwr https://windows.spacetimedb.com -useb | iex
```

Verify the installation:

```bash
spacetime version
```

---

## 2. Create Your Module

Initialize a new SpacetimeDB module in the language of your choice:

```bash
# TypeScript
spacetime init --lang typescript fantasy-football

# C#
spacetime init --lang csharp fantasy-football

# Rust
spacetime init --lang rust fantasy-football
```

This creates the project scaffold inside `./fantasy-football/spacetimedb/`.

---

## 3. Define Your Tables

Open the generated module file and replace the boilerplate with the tables shown below.

### 3.1 Players Table

Stores every NFL player available for drafting.

```typescript
// TypeScript
import { table, AlgebraicType } from "@clockworklabs/spacetimedb-sdk";

export const Player = table(
  { name: "player", public: true },
  {
    player_id: AlgebraicType.createU32Type(),      // primary key
    name: AlgebraicType.createStringType(),
    position: AlgebraicType.createStringType(),    // QB, RB, WR, TE, K, DEF
    nfl_team: AlgebraicType.createStringType(),
    bye_week: AlgebraicType.createU8Type(),
  }
);
```

```csharp
// C#
[SpacetimeDB.Table(Name = "player", Public = true)]
public partial class Player
{
    [SpacetimeDB.PrimaryKey] public uint PlayerId;
    public string Name = "";
    public string Position = "";  // QB, RB, WR, TE, K, DEF
    public string NflTeam = "";
    public byte ByeWeek;
}
```

```rust
// Rust
#[spacetimedb::table(name = player, public)]
pub struct Player {
    #[primary_key]
    pub player_id: u32,
    pub name: String,
    pub position: String,   // "QB" | "RB" | "WR" | "TE" | "K" | "DEF"
    pub nfl_team: String,
    pub bye_week: u8,
}
```

### 3.2 Fantasy Teams Table

One row per fantasy team (user) in a league.

```rust
// Rust example — adapt the pattern for C# / TypeScript
#[spacetimedb::table(name = fantasy_team, public)]
pub struct FantasyTeam {
    #[primary_key]
    pub team_id: u32,
    pub owner_identity: spacetimedb::Identity,  // SpacetimeDB identity of the user
    pub league_id: u32,
    pub team_name: String,
    pub total_points: f32,
}
```

### 3.3 Roster Table

Tracks which player is on which fantasy team and in which slot.

```rust
// Slot constants — use these everywhere to keep values consistent
pub const SLOT_STARTER: &str = "STARTER";
pub const SLOT_BENCH:   &str = "BENCH";
pub const SLOT_IR:      &str = "IR";

#[spacetimedb::table(name = roster, public)]
pub struct Roster {
    #[primary_key]
    pub roster_id: u32,
    #[index(btree)]
    pub team_id: u32,
    #[index(btree)]
    pub player_id: u32,
    pub slot: String,   // SLOT_STARTER | SLOT_BENCH | SLOT_IR
}
```

### 3.4 Live Scores Table

Holds real-time fantasy points accumulated by each rostered player for the current week.

```rust
#[spacetimedb::table(name = live_score, public)]
pub struct LiveScore {
    #[primary_key]
    pub player_id: u32,
    pub week: u8,
    pub points: f32,
    pub last_updated: spacetimedb::Timestamp,
}
```

### 3.5 Draft Pick Table

Records every pick made during the draft.

```rust
#[spacetimedb::table(name = draft_pick, public)]
pub struct DraftPick {
    #[primary_key]
    pub pick_number: u32,
    #[index(btree)]
    pub league_id: u32,
    pub team_id: u32,
    #[index(btree)]
    pub player_id: u32,
    pub picked_at: spacetimedb::Timestamp,
}
```

---

## 4. Write Your Reducers

Reducers are transactional functions that modify the database. Clients call them instead of writing SQL.

### 4.1 Draft a Player

```rust
#[spacetimedb::reducer]
pub fn draft_player(
    ctx: &spacetimedb::ReducerContext,
    league_id: u32,
    player_id: u32,
    pick_number: u32,
) -> Result<(), String> {
    // Use the (league_id, player_id) index to check availability in O(log n)
    let already_drafted = ctx
        .db
        .draft_pick()
        .league_id()
        .filter(&league_id)
        .any(|pick| pick.player_id == player_id);

    if already_drafted {
        return Err(format!("Player {} is already drafted", player_id));
    }

    // Find the calling user's fantasy team in this league
    let team = ctx
        .db
        .fantasy_team()
        .iter()
        .find(|t| t.league_id == league_id && t.owner_identity == ctx.sender)
        .ok_or("You don't have a team in this league")?;

    ctx.db.draft_pick().insert(DraftPick {
        pick_number,
        league_id,
        team_id: team.team_id,
        player_id,
        picked_at: ctx.timestamp,
    });

    // roster_id combines league and pick number to ensure global uniqueness.
    // Assumes leagues have at most 10 000 picks (typical: 12 teams × ~20 rounds = 240).
    // For production, prefer an auto-increment sequence or UUID instead.
    debug_assert!(pick_number < 10_000, "pick_number must be < 10_000");
    let roster_id = league_id
        .checked_mul(10_000)
        .and_then(|v| v.checked_add(pick_number))
        .ok_or("roster_id overflow — use a larger ID space for this league")?;
    ctx.db.roster().insert(Roster {
        roster_id,
        team_id: team.team_id,
        player_id,
        slot: SLOT_BENCH.to_string(),
    });

    Ok(())
}
```

### 4.2 Update Live Score (called from your score-ingestion service)

```rust
#[spacetimedb::reducer]
pub fn update_live_score(
    ctx: &spacetimedb::ReducerContext,
    player_id: u32,
    week: u8,
    points: f32,
) {
    if let Some(mut score) = ctx.db.live_score().player_id().find(&player_id) {
        score.points = points;
        score.last_updated = ctx.timestamp;
        ctx.db.live_score().player_id().update(score);
    } else {
        ctx.db.live_score().insert(LiveScore {
            player_id,
            week,
            points,
            last_updated: ctx.timestamp,
        });
    }
}
```

### 4.3 Move Player to Starter Slot

```rust
#[spacetimedb::reducer]
pub fn set_starter(
    ctx: &spacetimedb::ReducerContext,
    player_id: u32,
    team_id: u32,
) -> Result<(), String> {
    // Use the player_id index then filter by team_id in O(log n)
    let mut entry = ctx
        .db
        .roster()
        .player_id()
        .filter(&player_id)
        .find(|r| r.team_id == team_id)
        .ok_or("Player not on this roster")?;

    entry.slot = SLOT_STARTER.to_string();
    ctx.db.roster().roster_id().update(entry);
    Ok(())
}
```

---

## 5. Build and Publish Your Module

```bash
cd fantasy-football

# Build
spacetime build

# Start a local SpacetimeDB instance (for development)
spacetime start

# Publish the module locally
spacetime publish --server local fantasy-football-dev
```

To deploy to SpacetimeDB Cloud:

```bash
spacetime login
spacetime publish fantasy-football-prod
```

---

## 6. Connect Your Client

### JavaScript / TypeScript (React, Vue, Svelte, etc.)

Install the SDK:

```bash
npm install @clockworklabs/spacetimedb-sdk
```

Generate client-side type bindings from your published module:

```bash
spacetime generate --lang typescript --out-dir src/module_bindings
```

Connect and subscribe:

```typescript
import { SpacetimeDBClient } from "@clockworklabs/spacetimedb-sdk";
import { Player, Roster, LiveScore } from "./module_bindings";

const client = new SpacetimeDBClient(
  "ws://localhost:3000",   // or your cloud URI
  "fantasy-football-dev"
);

client.subscribe([
  "SELECT * FROM player",
  "SELECT * FROM roster",
  "SELECT * FROM live_score",
  "SELECT * FROM fantasy_team",
  "SELECT * FROM draft_pick",
]);

// React to live score changes in real time
LiveScore.onUpdate((oldScore, newScore) => {
  console.log(
    `Player ${newScore.playerId} now has ${newScore.points} points (week ${newScore.week})`
  );
});

// Draft a player from the client — handle success and error
client
  .call("draft_player", [leagueId, playerId, pickNumber])
  .then(() => console.log("Draft pick submitted"))
  .catch((err: Error) => console.error("Draft failed:", err.message));
```

### C# (Unity or .NET)

```csharp
using SpacetimeDB;
using SpacetimeDB.Types;  // generated bindings

var conn = await DbConnection.Builder()
    .WithUri("ws://localhost:3000")
    .WithModuleName("fantasy-football-dev")
    .Build();

conn.Db.LiveScore.OnUpdate += (oldScore, newScore) =>
{
    Debug.Log($"Player {newScore.PlayerId}: {newScore.Points} pts");
};

await conn.SubscribeAllTablesAsync();

// Draft a player — listen for reducer errors
conn.Reducers.OnDraftPlayerEvent += (reducerEvent, leagueId, playerId, pickNumber) =>
{
    if (reducerEvent.Status is UpdateStatus.Failed failed)
        Debug.LogError($"Draft failed: {failed.Message}");
    else
        Debug.Log("Draft pick submitted");
};

Reducer.DraftPlayer(leagueId, playerId, pickNumber);
```

---

## 7. Real-Time Score Ingestion

SpacetimeDB does not have direct internet access from within reducers. Use a thin external service to pull scores from your stats provider (e.g., ESPN API, Sleeper, NFL API) and push them in via the `update_live_score` reducer:

```bash
# Example using the CLI
spacetime call fantasy-football-prod update_live_score '[player_id, week, points]'
```

Or from a Node.js ingestion script:

```typescript
import { SpacetimeDBClient } from "@clockworklabs/spacetimedb-sdk";

const ingester = new SpacetimeDBClient("wss://maincloud.spacetimedb.com", "fantasy-football-prod");
await ingester.connect();

// Poll your stats API every 30 s during live games
setInterval(async () => {
  const scores = await fetchCurrentScores(); // your stats API call
  for (const s of scores) {
    ingester.call("update_live_score", [s.playerId, s.week, s.points]);
  }
}, 30_000);
```

All connected fantasy-app clients will receive the updated scores **automatically** because they subscribed to the `live_score` table.

---

## 8. Project Layout Summary

```
fantasy-football/
├── spacetimedb/           # SpacetimeDB module (server logic)
│   ├── src/
│   │   └── lib.rs         # (or index.ts / Lib.cs)
│   └── Cargo.toml         # (or package.json / .csproj)
├── client/                # Your front-end app
│   ├── src/
│   │   ├── module_bindings/   # auto-generated by spacetime generate
│   │   └── App.tsx
│   └── package.json
└── ingester/              # Score-ingestion microservice
    └── index.ts
```

---

## 9. Next Steps

- **Authentication**: SpacetimeDB assigns every connected client a cryptographic `Identity`. Use it (as shown in `draft_player`) to enforce ownership rules without any extra auth service.
- **Leagues & Commissioners**: Add a `league` table and restrict sensitive reducers (e.g., `update_live_score`) to the commissioner's identity or a service account.
- **Trades**: Create a `trade_proposal` table and a pair of `propose_trade` / `accept_trade` reducers that atomically swap players between rosters.
- **Playoffs & Scheduling**: Add a `matchup` table and a `schedule_matchups` reducer to automate bracket generation.
- **Deployment**: See the [SpacetimeDB Cloud docs](https://spacetimedb.com/docs) for region selection, scaling options, and CI/CD workflows.

---

## Resources

| Resource | Link |
|---|---|
| SpacetimeDB Documentation | https://spacetimedb.com/docs |
| CLI Reference | https://spacetimedb.com/docs/cli-reference |
| Rust Module Quickstart | https://spacetimedb.com/docs/quickstarts/rust |
| C# Module Quickstart | https://spacetimedb.com/docs/quickstarts/c-sharp |
| TypeScript Module Quickstart | https://spacetimedb.com/docs/quickstarts/typescript |
| Discord Community | https://discord.gg/spacetimedb |
