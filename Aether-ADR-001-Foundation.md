# ADR-001: Aether Platform Foundation

**Status:** Accepted
**Date:** 2026-07-16
**Owners:** Alkis (Lead Architect)
**Supersedes:** N/A — this is the founding document

---

## 1. Context

Aether began as a Python trading bot and has, over months of iteration (RL
observation design, WebSocket/market-data integration, DuckDB vs. TimescaleDB
storage debates, AI-assisted signal validation), repeatedly run into the same
wall: every new feature exposes a structural limitation, not a logic bug.
That pattern is the signal that the project needs a platform-grade foundation
before any further trading logic is written.

This ADR freezes the vision. It is the "constitution" every future module,
service, and ADR must comply with. Nothing here describes *how* a strategy
makes money — it describes the operating system that strategies, brokers,
and AI components will plug into.

---

## 2. Decision

### 2.1 What is Aether?

Aether is a **distributed, event-driven trading platform** — not a trading
bot, not a monolith. It is best understood as an *operating system for
trading*: a set of independent services that communicate exclusively through
a shared event bus and a shared contract language (typed messages). Strategies,
brokers, data providers, and AI/RL components are all **plugins** running on
top of this OS. None of them know whether they're operating against
historical data, a paper account, or a live market — that distinction is the
platform's job, not theirs.

### 2.2 Core Principles (binding on every service)

1. **Event-driven first.** Services communicate via NATS JetStream using
   Commands, Events, or Queries (§2.4) — never direct in-process calls across
   service boundaries, never shared mutable state.
2. **Uniform data source.** A strategy consumes `MarketTick`/`Signal`-shaped
   events regardless of whether they originate from Mnemosyne (backtest),
   paper trading, or Hermes (live feed). Swapping data source must never
   require touching strategy code.
3. **Contracts before code.** No service is implemented until its inbound
   and outbound message schemas are defined in `shared/schemas`.
4. **Plugins, not forks.** Strategies, broker adapters, indicators, and even
   the backtester are loaded dynamically through Forge. Adding a new one
   must never require modifying core services.
5. **Async everywhere.** All I/O-bound service code is `asyncio`-native.
   Blocking calls (model inference, heavy compute) are pushed to worker
   pools/processes, never the event loop.
6. **Typed and validated.** All configuration and all messages are Pydantic
   v2 models. No dict-passing across service boundaries.
7. **Observable by default.** Every service exposes health checks, structured
   logs, and Prometheus metrics from day one — not bolted on later.
8. **Fail loud, degrade gracefully.** A single service crashing (e.g. Athena,
   the AI service) must never take down order execution or risk checks.
   Critical path (Hermes → Apollo → Themis → Ares → Atlas) must survive the
   loss of any non-critical service.
9. **Intelligence is a consumer, not the foundation.** RL and LLM reasoning
   (Athena, Odyssey) are added last, after the platform's plumbing is proven
   stable via a working non-AI vertical slice.

### 2.3 Service Boundaries

| Domain | Codename | Responsibility (one sentence) |
|---|---|---|
| Platform | **Aether** | The entire ecosystem of services and contracts described in this ADR. |
| Core / Orchestration | **Nexus** | Coordinates service lifecycle, startup/shutdown ordering, and cross-cutting orchestration — it does not contain trading logic. |
| AI | **Athena** | Produces AI/LLM-derived reasoning and signal validation, consumed by other services as advisory input only. |
| Scheduler | **Chronos** | Triggers time- and event-based automation (market open/close jobs, retraining windows, report generation). |
| Market Data | **Hermes** | Ingests and republishes live and historical market data as normalized events. |
| Order Execution | **Ares** | Routes validated orders to the correct broker adapter and tracks their lifecycle. |
| Risk Engine | **Themis** | Enforces risk and compliance rules before any order is allowed to execute. |
| Portfolio | **Atlas** | Owns the current state of positions, balances, and P&L. |
| Backtesting | **Mnemosyne** | Replays historical data through the same event contracts used in live trading. |
| Feature Engineering | **Daedalus** | Computes indicators/features from market data for strategies and AI to consume. |
| Strategy Engine | **Apollo** | Hosts strategy plugins; turns market/feature events into `Signal` events. |
| Storage | **Hestia** | Abstracts all database access (TimescaleDB, cache) behind a single interface. |
| Messaging | **Iris** | The NATS JetStream event bus itself and its schema/subject conventions. |
| Monitoring | **Argus** | Collects health and metrics from all services; drives alerting thresholds. |
| Reporting | **Clio** | Generates periodic reports and analytics from historical + portfolio data. |
| Notifications | **Echo** | Delivers alerts (Telegram/email) triggered by other services' events. |
| Configuration/Secrets | **Oracle** | Central configuration and secrets loading via pydantic-settings. |
| Plugin Loader | **Forge** | Dynamically discovers and loads strategy, broker, and indicator plugins. |
| Broker Adapters | **Poseidon** | Translates platform `OrderRequest`/`OrderStatus` contracts to/from broker-specific APIs. |
| Market Scanner | **Helios** | Scans the tradeable universe and emits candidate symbols/events. |
| RL Environment | **Odyssey** | Provides RL training/inference environments conforming to the platform's event contracts. |
| Decision Validator | **Nemesis** | Final gate that validates a `Signal`/`OrderRequest` against sanity/AI checks before Themis. |

### 2.4 Communication Patterns

Not everything communicates the same way. Three distinct patterns, each with
its own NATS subject namespace:

- **Commands** (imperative, expects a result, at-most-once intent):
  `cmd.execution.execute_order`, `cmd.execution.cancel_order`,
  `cmd.strategy.reload`
- **Events** (facts that already happened, broadcast, many subscribers):
  `evt.marketdata.tick_received`, `evt.strategy.signal_generated`,
  `evt.execution.order_filled`, `evt.risk.limit_exceeded`,
  `evt.portfolio.position_closed`
- **Queries** (request/response, read-only, no side effects):
  `qry.portfolio.get_portfolio`, `qry.execution.get_open_orders`,
  `qry.marketdata.get_latest_price`

Rule: a service **never** infers a query from an event stream if a direct
query subject exists — this keeps read paths cheap and avoids event-sourcing
creep before it's needed.

### 2.5 Mandatory Technologies

| Layer | Technology |
|---|---|
| GUI | PyQt6 |
| APIs | FastAPI |
| Async runtime | asyncio |
| Event bus | NATS JetStream |
| Configuration | pydantic-settings |
| Validation | Pydantic v2 |
| Historical DB | TimescaleDB |
| Cache | Redis (optional, added when a concrete need appears) |
| Metrics | Prometheus |
| Dashboards | Grafana |
| Logging | structlog + stdlib logging |
| Service management | systemd |
| Containers | Docker (per service, where it adds isolation value) |
| Reverse proxy | Nginx or Caddy |
| Testing | pytest |
| Dependency management | uv |
| Code quality | Ruff + mypy |
| Packaging | Hatchling or uv build |

Anything not on this list requires a follow-up ADR before adoption.

### 2.6 Domain Diagram (informal)

```
                        Aether Platform
                 +---------------------------+
                 |           Nexus           |
                 +---------------------------+
                    |          |          |
                 Hermes     Apollo      Athena
                (Market)   (Strategy)     (AI)
                    |          |          |
                 Themis      Ares       Atlas
                 (Risk)   (Execution) (Portfolio)
                              |
                           Hestia
                          (Storage)
                              |
                            Argus
                         (Monitoring)
```

If any box can't be explained in one sentence (§2.3), it is too broad and
should be split.

### 2.7 Repository Skeleton (structure only, no logic yet)

```
aether/
    services/
        nexus/
        hermes/
        apollo/
        ares/
        themis/
        atlas/
        athena/
        chronos/
        argus/
        hestia/
    shared/
        events/
        models/
        schemas/
        messaging/
        config/
        utils/
    plugins/
        brokers/
        strategies/
        indicators/
    infrastructure/
        docker/
        systemd/
        prometheus/
    docs/
    tests/
```

### 2.8 Founding Contracts (illustrative, to be formalized in `shared/schemas`)

- `MarketTick`: symbol, timestamp, bid, ask, last, volume, provider
- `Signal`: strategy, symbol, action, confidence, reason, features
- `OrderRequest`: broker, symbol, side, quantity, order_type

Every service that touches market data, signals, or orders speaks these
exact shapes. A service that needs additional fields extends via composition,
not by forking the base schema.

---

## 3. Build Order (binding sequence)

1. **Freeze the vision** — this document.
2. **Define the domains** — §2.3 / §2.6, one sentence per box.
3. **Choose communication patterns** — §2.4.
4. **Build the skeleton** — §2.7, folders only, no business logic.
5. **Define contracts** — §2.8, formalized as Pydantic models in `shared/schemas`.
6. **Build infrastructure before features** — config loading, logging, NATS
   connectivity, TimescaleDB connection, health checks, metrics, service
   lifecycle, graceful shutdown. No trading logic yet.
7. **Build one vertical slice** — Market Data → Signal → Risk Check → Paper
   Execution → Portfolio Update → Dashboard. This is the proof that the
   architecture works end-to-end.
8. **Add intelligence last** — RL (Odyssey), LLM reasoning (Athena),
   prediction validation (Nemesis), ensemble/online learning. AI consumes
   the platform; it does not define it.

No step may be skipped or reordered without a superseding ADR.

---

## 4. Consequences

**Positive**
- New strategies, brokers, or AI components can be added without touching
  core services (Nexus, Hermes, Ares, Themis, Atlas).
- Backtesting, paper trading, and live trading share one code path per
  strategy, eliminating a whole class of "works in backtest, breaks live" bugs.
- Failure of a non-critical service (Athena, Clio, Echo) cannot take down
  the order-execution critical path.
- The system can scale horizontally later (multiple Hermes instances for
  different data providers, multiple Apollo instances for different
  strategy pools) because nothing communicates in-process.

**Costs / Tradeoffs**
- Significantly more upfront engineering than a single-process bot —
  accepted deliberately, per the stated multi-year horizon.
- Operational complexity increases (NATS, TimescaleDB, systemd units,
  Prometheus/Grafana all need to be run and maintained on the VPS).
- Debugging a cross-service event flow requires good tracing/log
  correlation from day one (tracing strategy to be defined in a follow-up
  ADR before Phase 6 is considered complete).

---

## 5. Open Questions for Follow-up ADRs

- ADR-002: Event schema versioning and backward-compatibility policy for `shared/schemas`.
- ADR-003: Dependency injection approach for services (constructor-based vs. a DI framework).
- ADR-004: Secrets management mechanism under Oracle (file-based vs. vault-based).
- ADR-005: CI/CD pipeline shape (per-service vs. monorepo-wide pipelines).
- ADR-006: Docker vs. bare-systemd tradeoffs per service, and when a service graduates from systemd to a container.
- ADR-007: Tracing/correlation-ID propagation across NATS messages.
- ADR-008: Multi-machine scaling plan (when Aether outgrows a single VPS).
