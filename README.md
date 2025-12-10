# Oracle Integration & Price Feed System
High-Performance Oracle Aggregation for Perpetual Futures DEX

Reliable • Manipulation-Resistant • Real-Time • Distributed

# Table of Contents

Overview

Architecture

Features

Repository Structure

Solana Anchor Program

Rust Backend Service

Database Schema

Redis Caching

REST API

WebSocket Streams

Running the System

Adding New Trading Symbols

Health Monitoring

Failover & Redundancy

Testing

Security Considerations

Performance & Scaling

# Overview

This system provides real-time, high-reliability price data for a perpetual futures DEX.
It integrates Pyth and Switchboard, applies consensus algorithms, and publishes validated prices to backend services.

The module acts as the single source of truth for:

Mark price

Funding rate calculations

Liquidations

Risk engine

Price analytics

The design focuses on:

Sub-second latency

Oracle manipulation resistance

Multiple redundancy layers

Failover guarantees

Full auditability

# Architecture
```
               ┌──────────────────────────────┐
               │     Solana Mainnet RPC       │
               └───────────────┬──────────────┘
                               │
          ┌────────────────────┼──────────────────────────┐
          │                    │                          │
   ┌────────────┐     ┌────────────────┐       ┌──────────────────┐
   │  Pyth Feed  │     │ Switchboard    │       │  Other Oracles   │
   └─────┬───────┘     └──────┬────────┘       └──────────────────┘
         │                     │
         ▼                     ▼
     ┌────────────────────────────────────────┐
     │    Anchor Oracle Smart Contract        │
     │   - read pyth price                    │
     │   - read switchboard                   │
     │   - validate consensus                 │
     └───────────────┬────────────────────────┘
                     │ RPC
                     ▼
     ┌────────────────────────────────────────┐
     │      Rust Oracle Backend Service       │
     │----------------------------------------│
     │ OracleManager                          │
     │ PythClient                             │
     │ SwitchboardClient                      │
     │ PriceAggregator                        │
     │ Redis Publisher                        │
     │ Postgres Logger                        │
     │ Health Monitor                         │
     └───────────────┬────────────────────────┘
                     │
         ┌───────────┼─────────────────────┐
         │           │                     │
     ┌──────────┐  ┌──────────┐     ┌──────────────────────┐
     │   Redis   │  │ Postgres │     │ REST & WS API Server │
     └──────────┘  └──────────┘     └──────────────────────┘
```
# Features
Oracle Integration

Pyth V2/V3 price feeds

Switchboard aggregators

Extensible to Chainlink, DIA, Band, Internal VWAP

Consensus & Safety

Median aggregation

Confidence weighting

Max deviation checks

Staleness enforcement

Outlier rejection

Flash crash detection

Backend

Real-time subscriptions

Circuit-breaker failover

Redis caching (<1ms reads)

Postgres historical storage

Fully typed REST API

WebSocket price pushes

Performance

≤500ms end-to-end latency

1000+ price queries per second

50+ symbols live

# Repository Structure
```
oracle-system/
│
├── AnchorProgram/           # Solana program
│   ├── Anchor.toml
│   └── programs/oracle_program/src/lib.rs
│
├── oracle-backend/         # Rust backend service
│   ├── src/
│   │   ├── clients/        # Pyth + Switchboard
│   │   ├── manager/        # OracleManager
│   │   ├── aggregator/     # PriceAggregation logic
│   │   ├── api/            # REST & WebSocket APIs
│   │   ├── storage/        # Redis & Postgres
│   │   └── models/
│   └── migrations/         # Database schema
│
├── docker-compose.yml
└── README.md
```
# Solana Anchor Program

The smart contract provides:

1. get_pyth_price()

Reads and validates a Pyth price account.

2. get_switchboard_price()

Reads Switchboard aggregator accounts.

3. validate_price_consensus()

Applies:

Median-based consensus

Max deviation filtering

Timestamp validation

Confidence checks

Returned prices are sent to backend for aggregation.

# Rust Backend Service
Core modules

PythClient – reads Pyth via RPC

SwitchboardClient – parses aggregator rounds

PriceAggregator – calculates final consensus price

OracleManager – orchestrates fetching & validation

RedisStore – caching layer

PostgresStore – persistent history

REST API – external price access

WebSocket API – real-time streams

Data Flow:
RPC → Clients → Aggregator → Redis → API → Consumers
                          ↘︎ Postgres

# Database Schema

Primary tables:

Table	Purpose
price_history	Final consensus price per symbol
source_prices	Raw Pyth/Switchboard price inputs
oracle_health	Track last update, staleness, status
deviation_alerts	Large deviation (>1% threshold)
manipulation_events	Suspicious behavior logs
aggregator_log	Audit trail of aggregation decisions
symbols	Registered trading pairs

Partitioning recommended for price_history and source_prices.

# Redis Caching
```
Keys:

oracle:price:{symbol}
oracle:confidence:{symbol}
oracle:timestamp:{symbol}
```
Cache usage:

Liquidation engine

Funding rate calculator

Frontend UI

Market data pipelines

Avg read time: 0.2ms – 1ms

# REST API
GET /oracle/price/:symbol
```
{
  "symbol": "BTC",
  "data": {
    "price": 63245.2,
    "confidence": 0.5,
    "timestamp": 1712332143
  }
}
```
GET /oracle/prices

Returns all symbols.

GET /oracle/history/:symbol?limit=500

Paginated history.

GET /oracle/sources/:symbol

Raw Pyth/Switchboard inputs.

GET /oracle/health

Staleness + reliability metrics.

# WebSocket Streams
/ws/prices/{symbol}

Real-time price updates.

/ws/alerts

Oracle manipulation alerts.

/ws/health

Source reliability stream.

# Running the System
1. Install dependencies

Rust (1.75+)

Anchor (0.29+)

Solana (1.18+)

Docker & Docker Compose

2. Start Postgres + Redis
```
docker-compose up -d
```
3. Deploy Anchor program
```
cd AnchorProgram
anchor build
anchor deploy
```
4. Start Rust backend
```
cd oracle-backend
cargo run
```
REST API starts on http://localhost:8080.

# Adding New Trading Symbols

Insert into the symbols table:
```
INSERT INTO symbols(symbol, pyth_feed, switchboard_feed)
VALUES ('SOL', 'pyth_pubkey_here', 'switchboard_pubkey_here');
```

Add symbol to backend Config (yaml or json):
```
{
  "symbol": "SOL",
  "pyth": "xxx",
  "switchboard": "yyy"
}

```
Service automatically starts streaming updates.

# Health Monitoring

The backend tracks:

Oracle staleness

Confidence widening

Consecutive update failures

RPC timeouts

Large price deviations

Flash crashes (rate-of-change detection)

Statuses:

healthy

degraded

offline

# Failover & Redundancy
Normal mode:

Pyth + Switchboard → Median → Redis

If one oracle goes offline:

Only remaining oracle used (confidence penalty applied)

If both offline:

System enters stale mode

Last known price held temporarily

Liquidation system notified

Alerts broadcast via WebSocket

# Testing
Unit Tests

Price parsing

Confidence validation

Deviation checks

Median logic

Integration Tests

Localnet Pyth & Switchboard feeds

RPC disconnect scenarios

Stale data injection

Oracle manipulation simulations

Chaos Tests

Random network delays

Random oracle outages

Zero-price + extreme-price values

# Security Considerations

All oracle reads validated on-chain

Staleness enforced down to 30 seconds

Confidence bounds required

Outlier rejection

Prevention of price spoofing

Replay attack protection

RPC integrity checks

# Performance and Scaling

<500ms end-to-end latency target

1000+ price reads per second from Redis

Async parallel price fetching

Horizontal scaling via load-balanced backend instances

Historical storage compressed & partitioned
