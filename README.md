# 🔋 Smart Battery Management System — Database Backend

> A high-integrity, cloud-native PostgreSQL database backend for real-time battery monitoring, fault detection, and analytics — built on Supabase at zero cost.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Database Schema](#database-schema)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Analytics Queries](#analytics-queries)
- [Performance Benchmarks](#performance-benchmarks)
- [Tech Stack](#tech-stack)
- [Author](#author)

---

## Overview

This project implements a **Smart BMS (Battery Management System) Backend** using PostgreSQL on Supabase. It centralizes monitoring of battery packs, cell health, and charging cycles entirely within the database layer — offloading critical logic to PL/pgSQL triggers for real-time fault detection and stored procedures for intelligent charging profile recommendations.

The schema is designed in **Third Normal Form (3NF)** with **EER specialization** (Table-Per-Subclass) to support three distinct battery chemistries: NMC, LFP, and LTO.

---

## Features

- ✅ **18-table normalized schema** (3NF) with 5 lookup tables eliminating all transitive dependencies
- ⚡ **Automated fault detection** via PL/pgSQL trigger (`fn_fault_detection`) — executes in under 5ms per INSERT
- 🔌 **Intelligent charging profile recommendation** via stored procedure (`sp_recommend_charging_profile`) with 5 decision branches
- 🧬 **EER specialization** — `BatteryCell` superclass with disjoint `NMC_Cell`, `LFP_Cell`, `LTO_Cell` subclasses
- 📊 **7 advanced analytics queries** using CTEs and Window Functions (RANK, LAG, FIRST_VALUE, moving averages)
- 🗂️ **BRIN + partial indexes** for time-series performance optimization
- 🔗 **Cascade delete integrity** — removing a BatteryPack automatically cleans all dependent records

---

## Database Schema

### Entity Overview

| Table | Description |
|---|---|
| `BatteryPack` | Top-level battery pack (EV, Grid Storage, Portable, Industrial) |
| `BatteryCell` | Individual cells linked to a pack; superclass for chemistry subclasses |
| `NMC_Cell` | NMC-specific attributes (nickel/manganese/cobalt ratios, energy density) |
| `LFP_Cell` | LFP-specific attributes (iron phosphate purity, thermal stability rating) |
| `LTO_Cell` | LTO-specific attributes (titanate grade, rated cycle life) |
| `Sensor` | Voltage, Current, or Temperature sensors attached to cells |
| `SensorReading` | Time-series sensor measurements (BIGSERIAL, BRIN-indexed) |
| `ChargingProfile` | Chemistry-specific charging profiles with SoC/SoH range constraints |
| `ChargeCycle` | Individual charge/discharge cycle records |
| `BatteryHealthLog` | Periodic SoC, SoH, and capacity snapshots |
| `FaultLog` | Auto-populated fault records with severity and deviation percentage |
| `SensorReadingSummary` | Hourly aggregated summaries per sensor |
| `ApplicationType` | Lookup: EV, Grid_Storage, Portable, Industrial |
| `ChemistryType` | Lookup: NMC, LFP, LTO |
| `CellStatus` | Lookup: Active, Retired |
| `FaultTypeLookup` | Lookup: 7 fault types (Over_Voltage, Over_Temperature, etc.) |
| `FaultSeverityLookup` | Lookup: Warning / Critical / Emergency |

### Fault Severity Logic

| Severity | Deviation from Threshold |
|---|---|
| Warning | ≤ 5% |
| Critical | 5% – 15% |
| Emergency | > 15% |

---

## Project Structure

```
smart-bms-database/
│
├── schema/
│   ├── 00_reset.sql              # Drop all tables/functions (run first)
│   ├── 01_lookup_tables.sql      # Reference/lookup tables
│   ├── 02_core_tables.sql        # Main schema + indexes
│
├── data/
│   ├── insert_application_type.sql
│   ├── insert_chemistry_type.sql
│   ├── insert_cell_status.sql
│   ├── insert_fault_type_lookup.sql
│   ├── insert_fault_severity_lookup.sql
│   ├── insert_battery_pack.sql
│   ├── insert_battery_cell.sql
│   ├── insert_nmc_cell.sql
│   ├── insert_lfp_cell.sql
│   ├── insert_lto_cell.sql
│   ├── insert_sensor.sql
│   ├── insert_sensor_reading.sql
│   ├── insert_charging_profile.sql
│   ├── insert_charge_cycle.sql
│   ├── insert_battery_health_log.sql
│   ├── insert_fault_log.sql
│
├── programming/
│   ├── trigger_fault_detection.sql    # fn_fault_detection() + CREATE TRIGGER
│   ├── sp_charging_profile.sql        # sp_recommend_charging_profile()
│
├── analytics/
│   ├── Q1_soh_degradation_trend.sql
│   ├── Q2_abnormal_soh_drop.sql
│   ├── Q3_chemistry_durability_benchmark.sql
│   ├── Q4_fast_vs_standard_charging.sql
│   ├── Q5_fault_frequency_by_chemistry.sql
│   ├── Q6_end_of_life_detection.sql
│   ├── Q7_rolling_7day_sensor_average.sql
│
├── erd/
│   └── erd_diagram.png               # EER Model diagram
│
└── README.md
```

---

## Getting Started

### Prerequisites

- A [Supabase](https://supabase.com) account (free tier is sufficient)
- No local PostgreSQL installation required

### Deployment Steps

**Step 1 — Reset (clean slate)**

In your Supabase project, open the **SQL Editor** and run `schema/00_reset.sql` to drop any existing objects in the correct dependency order.

**Step 2 — Create schema**

Run `schema/01_lookup_tables.sql` followed by `schema/02_core_tables.sql`. You should see 18 tables listed in the Table Editor sidebar.

Verify with:
```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

**Step 3 — Load seed data**

Run all 16 files in the `data/` folder in order. This inserts: 3 battery packs, 6 cells (2 NMC + 2 LFP + 2 LTO), 18 sensors, 8 charging profiles, 8 charge cycles, 17 health log entries, and 2 historical fault records.

**Step 4 — Deploy trigger and stored procedure**

Run `programming/trigger_fault_detection.sql` then `programming/sp_charging_profile.sql`.

Test the trigger by inserting an over-voltage reading (e.g., 4.250V for an NMC cell with max 4.200V) — a fault row should appear automatically in `FaultLog`.

Test the stored procedure:
```sql
SELECT * FROM sp_recommend_charging_profile(1);   -- NMC, SoH ~82% → RECOMMENDED
SELECT * FROM sp_recommend_charging_profile(5);   -- LTO, SoH ~99.4% → RECOMMENDED
SELECT * FROM sp_recommend_charging_profile(999); -- Non-existent → ERROR
```

**Step 5 — Run analytics queries**

Execute each file in `analytics/` individually in the SQL Editor.

---

## Analytics Queries

| Query | Description | Techniques Used |
|---|---|---|
| Q1 | SoH Degradation Trend | `FIRST_VALUE`, `LAG`, moving `AVG OVER ROWS` |
| Q2 | Abnormal SoH Drop Detection (> 3%) | `LAG`, date arithmetic |
| Q3 | Cross-Chemistry Durability Benchmark | `RANK()`, CTEs |
| Q4 | Fast vs Standard Charging Impact | CTE, `STDDEV`, `GROUP BY` |
| Q5 | Fault Frequency by Chemistry & Severity | Window `SUM`, running totals |
| Q6 | End-of-Life Cell Detection (SoH < 80%) | CTE, `DISTINCT ON` |
| Q7 | Rolling 7-Day Average Sensor Readings | `AVG OVER ROWS BETWEEN` |

---

## Performance Benchmarks

| Metric | Result |
|---|---|
| Trigger execution overhead | < 5 ms per INSERT |
| Fault detection decision time | < 1 ms |
| Analytics queries (Q1–Q7) | 50–300 ms |
| Schema creation (18 tables) | < 2 seconds |
| Stored procedure response time | < 10 ms |
| Time-series index type | BRIN (space-efficient) |

---

## Tech Stack

| Tool | Purpose |
|---|---|
| PostgreSQL 15 | Relational database engine |
| Supabase (Free Tier) | Hosted PostgreSQL backend + SQL Editor |
| PL/pgSQL | Trigger functions and stored procedures |
| pgAdmin | Query execution and schema verification |
| Git / GitHub | Version control |

**Total project cost: PKR 0** — all tools used are open-source or free tier.

---

## Author

**Syed Ammar Zulfiqar** — Roll No. 22K-4845  
Department of Electrical Engineering  
FAST-NUCES, Karachi — Spring 2026  
Course: Fundamentals of Database (CS-2011)  
Submitted to: Dr. Ahsan Nadeem
