# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: Uses OPENROUTER_MODEL from .env (default: openrouter/openrouter/free), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start.sh              # Launch Docker container (Linux)
│   ├── stop.sh               # Stop Docker container (Linux)
│   ├── start_mac.sh          # Launch Docker container (macOS)
│   ├── stop_mac.sh           # Stop Docker container (macOS)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=true
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM defined, requesting structured output
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Scripts

**`scripts/start.sh`** (Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop.sh`** (Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_mac.sh`** (macOS):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

**`scripts/test.sh`** (Linux / macOS)
- Run all automated tests by default, launching container if needed
- If name of tests given, run only that specific test
- if any container started, stop in the end and clean what was used from this container (docker system prune)

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Review Notes — Questions, Clarifications & Simplification Opportunities

_Added during a plan review. The market data subsystem is already implemented (`backend/app/market/`, summarized in `planning/MARKET_DATA_SUMMARY.md`); several items below are gaps between that implementation and this plan. Resolve the blocking questions and fold the answers into the sections above; this section can then be deleted or kept as a decision log._

The five items most likely to cause divergent agent implementations are: the LLM model configuration, the undefined "daily change %", the SSE payload shape, the active ticker set, and the trade/chat response contracts.

### 13.1 Blocking questions (answer before implementation)

1. **LLM configuration is inconsistent and the default model ID looks invalid.**
   - §5 says `OPENROUTER_API_KEY` is required, but `LLM_MOCK=true` is supposed to work without a key. Clarify: "required unless `LLM_MOCK=true`".
   
   Answer:
   - Removed blockers not relevant. 
   - Correct. LLM_MOCK=true will force usage of seed data.

2. **"Daily change %" has no data source.**
   - `PriceUpdate` (`backend/app/market/models.py`) tracks the last tick's `previous_price` and computes `change_percent` against it; it has no previous-close/daily-open field. The simulator has no previous close or daily open; Massive may provide one, but the cache does not store it.
   - Define the watchlist column: (a) change since page load/session start (rename it "session change %"), (b) change vs previous close (add `prev_close`/`day_open` to `PriceUpdate` and populate it in both sources), or (c) drop it and show only the flash/direction.
   - Recommendation: (a) for v1; it matches the sparkline "since page load" behavior and needs no data-model change.
   
3. **Pin the SSE contract; it differs from the existing implementation.**
   - §6 says each event contains ticker, price, previous price, timestamp, and direction (per-ticker events). `backend/app/market/stream.py:75-83` sends one unnamed `data:` event containing a JSON object keyed by ticker, only when the cache version changes, with `retry: 1000` and no heartbeat.
   - Document the exact shape, event name (or none), initial snapshot, update cadence, and keepalive (e.g., a comment line every 15s). Frontend and tests must code to one contract.
   - Recommendation: keep the existing map-per-event shape and add a heartbeat.

4. **Which tickers are subscribed to?**
   - §6 says "all tickers known to the system — in the single-user model this is equivalent to the user's watchlist." That breaks when a ticker is held in `positions` but removed from the watchlist, or bought without ever being watched: its price stops updating and valuation/P&L goes stale.
   - Define the active set as `watchlist ∪ positions` (plus the selected ticker if charts should work for unwatched tickers), and recompute it after every watchlist/trade change.
   - `remove_ticker` in `backend/app/market/interface.py:48-53` also removes the ticker from `PriceCache`. Do not call it while a position exists.
   - Recommendation: a small `refresh_subscriptions()` service helper owns this union; API handlers never call the market source directly.

5. **When is a ticker price available, and which tickers are valid?**
   - The trade bar and watchlist accept arbitrary symbols. The simulator seeds unknown tickers at a random `$50–$300` (`backend/app/market/simulator.py:151`); Massive may return no snapshot for an invalid symbol.
   - Define ticker normalization (uppercase/trim), format validation, behavior when no price exists yet (reject with a clear error vs queue/retry), and whether unknown tickers are allowed.
   - Recommendation: allow arbitrary well-formed symbols in simulator mode, but require a cached price before executing a trade; in Massive mode show "no data" until the first quote and reject trades without a price. Optionally restrict the simulator to a known universe of ~30 tickers for predictability.

6. **What do the trade and chat responses return, and how does the UI stay in sync?**
   - SSE only streams prices. Cash, positions, and watchlist changes are not pushed. After `POST /api/portfolio/trade` or `POST /api/chat`, the frontend needs the updated portfolio and possibly watchlist.
   - Define response shapes. Recommendation: trade returns `{trade, portfolio}`; chat returns `{message, actions, portfolio, watchlist}` (or a `refresh_required` flag); alternatively the frontend refetches `/api/portfolio` and `/api/watchlist` after every mutation.
   - Define the error contract: FastAPI `detail` string vs structured `{code, message, field}`; status codes for validation (400/422), unknown ticker (404), duplicate (200/409), insufficient cash/shares (400/422).

7. **Pin the portfolio math and rounding rules.**
   - Define exactly: `total_value = cash_balance + Σ(quantity × latest_price)`; `unrealized_pnl = (latest_price − avg_cost) × quantity`; `pnl_pct = (latest_price − avg_cost) / avg_cost × 100` (state the denominator).
   - Average-cost method: weighted average on buys, unchanged on sells; no realized P&L display; delete the row when quantity reaches zero.
   - Rules: no shorting; sell quantity ≤ held; buy cost ≤ cash; quantity > 0; fractional precision (e.g., 4–6 decimals); cash rounded to cents. State where rounding happens (per trade vs display) so tests are deterministic.
   - Market orders fill at the latest cached price. Define stale-price behavior for Massive (market closed/weekend): accept last price, or reject if older than N seconds/minutes? Recommendation: accept and show the timestamp for the demo.
   - Define `portfolio_snapshots` retention and the `GET /api/portfolio/history` default window/limit; record an initial snapshot at startup so the chart is not empty.

8. **The database path/volume is ambiguous.**
   - §4 has `backend/db/` for schema/seed logic and a top-level `db/` for the runtime SQLite file. Two directories named `db` will confuse agents.
   - §11 uses `-v finally-data:/app/db` (a named volume), then says "The `db/` directory in the project root maps to `/app/db`" (a bind mount). Pick one.
   - Recommendation: rename `backend/db/` → `backend/schema/` (or `backend/database/`); use the named volume for the one-command path and remove the top-level `db/` + `.gitkeep`, or use a bind mount and document permissions. If you keep a named volume, `db/finally.db` never appears in the repo, which is fine.
   - Clarify the container path (`/app/db`) vs repo path (`db/`) in the Dockerfile and scripts.

9. **Startup, initialization, and worker model.**
   - §7 says "on startup (or first request)" and "lazily initializes the database." Choose one. Recommendation: initialize in the FastAPI `lifespan` handler (create tables + seed if empty), then start the market source and snapshot task; cancel both on shutdown. Lazy first-request init adds a race and hides startup failures.
   - Run exactly **one** uvicorn worker. Multiple workers would start multiple simulators/pollers, duplicate background snapshots, split SSE clients, and contend on SQLite. Document this in the Docker `CMD` and in the plan.
   - SQLite: enable WAL + `busy_timeout`, use one connection per request (or a small pool), and keep writes short. State whether you use `aiosqlite`, `sqlite3` in a thread, or SQLAlchemy async. Recommendation: plain `aiosqlite`/SQL + Pydantic; no ORM/Alembic for this scope.

10. **E2E determinism is underspecified.**
    - The simulator uses global `random` and `np.random`; tests that assert flashing/streaming can be flaky.
    - Add a deterministic mode: `MARKET_DATA_SEED` (seed both RNGs) and/or a fixed-tick source for E2E. Keep Massive out of E2E.
    - Define `LLM_MOCK=true` precisely: what canned message is returned, how a test triggers a trade/watchlist action, whether there is a simulated delay (so the loading state can be asserted), and where the mock lives. Recommendation: a rule-based mock that recognizes `BUY <qty> <ticker>` / `SELL <qty> <ticker>` / `WATCH <ticker>` and otherwise returns a fixed analysis; zero delay by default, optional `LLM_MOCK_DELAY_MS`.
    - `docker-compose.test.yml`: use a fresh/tmpfs DB volume per run, wait for `/api/health`, and use a fixed or dynamically allocated port. Avoid relying on browser auto-open.
    - SSE reconnection is hard to test reliably in Playwright. Consider a backend integration test with a raw SSE client, or use Playwright's offline toggle and assert the status dot returns to green.

Answers: 
- Everything from 2 to 10 will be fixed / implemented step by step and not at once.

### 13.2 Clarifications and contract details (non-blocking)

- **Chat history window.** "Recent conversation history" will remain available within the same session. If tab or browser closes, or if session id change, history will be cleaned up.  
- **`chat_messages.actions` shape.** Define the JSON schema so the frontend can render confirmations and tests can assert. Example: `[{"type":"trade","ticker":"AAPL","side":"buy","quantity":10,"price":190.12,"status":"executed"}, {"type":"watchlist","ticker":"PYPL","action":"add","status":"executed"}]`. Include `status` and `error` for failures; clarify whether failed actions are stored.
- **Watchlist changes by the LLM.** The schema only shows `{"ticker":"PYPL","action":"add"}`. Confirm `remove` is supported, and whether the LLM can remove a ticker that is currently held (recommend: yes, remove from watchlist only; position and subscription remain).
- **Multiple LLM trades.** Execute in order against updated cash/positions; report per-action success/failure; do not abort the whole batch on one failure unless specified.
- **LLM failure handling.** Define timeout (e.g., 30s), retry policy (one retry on malformed JSON?), and user-facing fallback. Store the user message even if the LLM call fails? Recommendation: store it, return a friendly error, and do not fabricate an assistant message.
- **Massive polling.** §6 says "configurable interval" but no env var exists (`backend/app/market/massive_client.py:32` hardcodes 15s). Add `MASSIVE_POLL_INTERVAL` (default 15s) and document behavior on 401/429/network errors (keep last prices, log, retry; consider backoff). Decide whether an invalid key fails fast or silently falls back to the simulator. Recommendation: fail fast with a clear log.
- **Massive ticker coverage.** Polling uses `get_snapshot_all(tickers=...)`; confirm the free-tier math (5 calls/min → 15s) and what happens when the watchlist grows. Consider a watchlist cap (e.g., 20 tickers) to bound API usage and UI density.
- **Stale/closed-market prices.** For Massive, prices are last trades and may be hours old after hours. Define whether the UI shows "last updated" and whether trades still fill at that price. Recommendation: yes for the demo; show the timestamp.
- **Connection status transitions.** EventSource retries indefinitely; a red state may never occur without a timer. Define: green on `onopen`, yellow on `onerror`/reconnecting, red after N failed attempts or M seconds without a message.
- **Sparkline vs main chart history.** Both are "since page load" today (sparkline explicitly; main chart implicitly). If the main chart should show more history, add a per-ticker ring buffer endpoint (`GET /api/prices/{ticker}/history`) or state the limitation. Recommendation: state "session-only charts" for v1.
- **Empty states.** No positions (heatmap), no snapshots (P&L chart), no chat history, no watchlist. Define placeholders so agents don't invent inconsistent UI.
- **Timezone/format.** Store timestamps as UTC ISO 8601; display local time in charts. Confirm the SSE `timestamp` remains Unix seconds (existing `PriceUpdate`) or switch to ISO; pick one and use it everywhere.
- **Data model wording.** "All tables include a `user_id` column" is not true for `users_profile` (its primary key is `id`). Reword or rename. Confirm whether `trades` stores failed trades (the schema has no `status`); recommendation: store successful fills only and return failures to the caller, or add `status`/`error` for auditability.
- **Development workflow.** The plan only describes production Docker. Add a local dev section: backend `uv run uvicorn app.main:app --reload --port 8000`; frontend `npm run dev` on 3000 with Next.js rewrites/proxy for `/api/*` and `/api/stream/*`, or enable CORS for `localhost:3000` in dev only.
- **Testing stack.** "React Testing Library or similar" leaves the choice open. Pick Vitest + React Testing Library if nothing else specified, and document the command. Backend tests should use a temp SQLite file and dependency override, not the runtime volume; add tests for duplicate watchlist adds, removing a held ticker (price still streams), missing/stale price trades, malformed LLM JSON, and multiple trades exceeding cash.
- **Security/ops.** No auth + a server-side OpenRouter key means public deployment lets anyone spend your credits and trade. Bind to `127.0.0.1` by default in the start scripts (`-p 127.0.0.1:8000:8000`), add a `.dockerignore` (`.env`, `db/*.db`, `node_modules`, `.next`, `__pycache__`, `.git`), never expose the key to the frontend, render chat text without `dangerouslySetInnerHTML`, and warn that public deployment needs auth/rate limiting.
- **Health check.** Add a Docker `HEALTHCHECK` and make `/api/health` verify DB + market source readiness, not just return `{"status":"ok"}`.
- **Browser auto-open.** §2 says "A browser opens"; §11 says "Optionally opens the browser." Define the default (auto-open, with `--no-open`) and make failure non-fatal (headless/remote environments).
- **Accessibility.** Add baseline expectations: labels on the trade inputs, keyboard focus states, sufficient contrast, and `prefers-reduced-motion` to disable price flashes. Don't rely on green/red alone; include arrows or signs.
- **Logging.** There will be no loggin on this solution for now.

### 13.3 Opportunities to simplify

2. **Use Vite + React instead of Next.js static export.** The UI is a single-page, client-rendered app with no SSR/SEO needs. Next.js adds `output: 'export'`, image/route constraints, and a heavier build for no benefit here. Vite + React + TypeScript + Tailwind is simpler and faster in Docker. If Next.js is a course requirement, keep it but add one sentence explaining why, and confirm no server features are used.
3. **Pick one charting approach.** The plan says "Canvas-based charting library preferred (Lightweight Charts or Recharts)" — Recharts is SVG, not canvas. For ~10 tickers and session-length data, Recharts alone (LineChart, Treemap) is sufficient and avoids a second library. If you want canvas, use Lightweight Charts for price charts and build the heatmap/treemap with CSS or a small component; don't mix both without a performance reason.
4. **Use one Docker entry point.** Four OS-specific scripts + an optional `docker-compose.yml` duplicate build/run/stop logic. Make `docker compose up -d --build` / `docker compose down` the single path, and keep `start_mac.sh`/`start_windows.ps1` as thin wrappers only if auto-open is required. Fewer paths to keep in sync and fewer Windows quoting bugs.
5. **Remove the top-level `db/` directory if you use a named volume.** With `-v finally-data:/app/db`, the repo's `db/` is unused. Keep it only for a bind mount. Pick one storage strategy and document it in one place.
6. **Rename `backend/db/` to `backend/schema/`.** This removes the two-`db` ambiguity and makes it obvious that one is code/SQL and the other is runtime data.
7. **Restrict the simulator universe (optional).** Arbitrary tickers get random seed prices and default volatility, which makes demos less realistic and tests less predictable. A fixed universe of ~30 tickers with known seed prices is simpler; if arbitrary tickers are required, document the random seed behavior and the Massive invalid-symbol behavior.
8. **Use plain SQL + Pydantic, not an ORM/migrations.** The plan already promises no migration step. Keep it that way: `CREATE TABLE IF NOT EXISTS` + idempotent seed in `lifespan`, `aiosqlite` (or `sqlite3` in a thread), and Pydantic response models. Avoid SQLAlchemy + Alembic for a single-file demo DB.
9. **Use a simple CSS/flex heatmap instead of a true treemap (optional).** For ~10 positions, a grid of cards with `flex-grow` proportional to weight, colored by P&L, is much simpler than a squarified treemap and still reads well. If the treemap look is important, use Recharts' `Treemap` and don't add a second library.
10. **Use inline SVG sparklines.** A small `<polyline>` component avoids pulling a chart library into every watchlist row. Reserve Recharts (or Lightweight Charts) for the main chart and P&L chart.
11. **Use a single `actions` array in the LLM schema.** A discriminated union (`{"type":"trade",...}` / `{"type":"watchlist",...}`) matches the persisted `actions` shape and reduces branching. If separate `trades`/`watchlist_changes` arrays are easier for the model, keep them but convert to one persisted shape.
12. **Use `INTEGER PRIMARY KEY` for append-only tables** (`trades`, `portfolio_snapshots`, `chat_messages`) instead of UUID `TEXT`. Smaller indexes, simpler ordering, no UUID dependency. Natural keys are enough for `watchlist`/`positions` (`(user_id, ticker)` or just `ticker`).
13. **Don't build a custom SSE reconnect state machine.** `EventSource` already reconnects. Map `onopen`/`onerror` to the status dot and avoid a separate polling health check in the frontend.
14. **Drop the optional Terraform/cloud deployment from the core plan.** Keep it as a one-line stretch goal; it should not shape the Dockerfile or add auth complexity to v1.

### 13.4 Minor consistency fixes

- Complete §9 step 4 ("Calls the LLM defined...").
- Add `OPENROUTER_MODEL` to §5 and `.env.example`, and fix the `openrouter/openrouter/free` default.
- Use `npm ci` (not `npm install`) and `uv sync --frozen` in Docker for reproducible builds; add `.dockerignore`.
- Confirm the chosen Next.js major supports Node 20, and pin both.
- Register API routes before the static-file catch-all in FastAPI; define the SPA fallback (`index.html`) explicitly.
- Clarify `.env` loading: `--env-file` injects env vars and does not mount the file. If the backend also reads `.env` directly for local dev, add `python-dotenv` and document precedence.
- Treat `LLM_MOCK` case-insensitively (`true`/`1`), and document it in `.env.example`.
- Add a `.env.example` entry for every variable mentioned in the plan (including any new `MARKET_DATA_SEED`, `MASSIVE_POLL_INTERVAL`, `LLM_MOCK_DELAY_MS`).
