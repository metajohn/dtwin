# Weather Machine — Digital Shadow Data Pipeline

## Project Status

This project is **complete and running in production**. The original local Python prototype has been superseded by a networked C#/.NET Azure implementation, which has been live and continuously collecting weather data since [DATE].

The project is not under active feature development — it served its purpose as a hands-on learning project for C#/.NET on Azure, built on top of an existing Python/Unreal prototype. Visual effects in the Unreal client are intentionally minimal (the focus was the data pipeline and client-server architecture, not the visualization layer).

## Overview

WeatherMachine is a "digital shadow" system that synchronizes real-world environmental data with a live 3D visualization in Unreal Engine 5. It pulls live weather data on a schedule, stores it, and serves it to a networked Unreal client that renders it as an interactive environment.

The project went through two architectures as requirements clarified:

1. A **local Python prototype**, using a file-based JSON middleware layer to bridge a script-driven data pipeline to Unreal Engine on the same machine.
2. A **networked C#/.NET production version**, once the local prototype validated the core data model and the requirements simplified enough to move to a proper client-server design over Azure.

Both are documented below for completeness.

---

## Production Architecture (C#/.NET on Azure)

The current, running implementation. Two Azure Functions and a networked Unreal Engine 5 C++ client.

### Components

- **`FetchLatestWeatherData`** — Timer-triggered Azure Function (hourly). Pulls live data from the OpenWeather API, transforms it into domain data (wind vector from speed/degrees, sun-elevation alpha from sunrise/sunset timestamps, ISO time conversion, weather-state classification), and persists it to Azure SQL via EF Core.
- **`GetWeatherById`** — HTTP-triggered Azure Function. A function-key-authenticated REST GET endpoint that returns either the latest record (no `id` param) or a specific historic record by ID, with `IsLive` computed dynamically based on the request.
- **Unreal Engine 5 C++ client** — Polls the endpoint on an interval it calculates itself from the server's last timestamp plus update interval, rather than a fixed poll loop. Includes connection-retry handling on failed requests and debounced network requests when a user scrubs through historic data.

### Key Details

- **Dynamic poll scheduling** — the client computes its next expected update time from the server's own timestamp and interval, rather than polling blindly, reducing unnecessary requests.
- **Debounced historic scrubbing** — rapid input while scrubbing through historic IDs is debounced (200ms) before triggering a network request.
- **Connection resilience** — failed requests trigger a bounded retry sequence rather than failing silently.

### Tech Stack

- Backend: C#, .NET, Azure Functions, Entity Framework Core, Azure SQL
- Client: Unreal Engine 5 (C++)
- Data Source: OpenWeather API

---

## Local Prototype (Python)

The original proof of concept. Validated the core data model and surfaced real concurrency problems before the architecture moved to a networked design.

### Local Technical Architecture

The system was designed as a multi-process pipeline to ensure data integrity and system stability:

- **Data Injection** — A Python "Pulse" script simulating environmental telemetry (temp, wind, solar alpha), logged to a time-series SQLite database.
- **The Bridge (Middleware)** — A Python service using `watchdog` to monitor file-system events, handling the handshake between Unreal Engine and the database.
- **State Management** — A bi-directional "JSON Handshake" managing live vs. historical data states.
- **Visualization** — An Unreal Engine 5 dashboard translating raw values into environmental effects (lighting, cloud density, wind physics).

### Key Features (Prototype-Only)

These solved real problems in the local, single-machine version. They were not carried forward into the networked production version, since the client-server redesign made them unnecessary — noted here as an accurate record of what was built and learned, not as claims about the current system.

- **Concurrency & Robust I/O** — Retry-with-delay logic to handle race conditions from multiple scripts reading/writing the same file.
- **Atomic Data Transactions** — Atomic file swapping (`os.replace`) so Unreal never read a corrupted or partial JSON packet.
- **Bi-Directional State Locking** — A history-lock protocol so Unreal could scrub through time without desyncing from the database.
- **Modular Data Normalization** — A unified `WeatherPacket` dataclass as a single source of truth across the pipeline.

### Tech Stack

- Engine: Unreal Engine 5 (Blueprints, Input)
- Language: Python 3.13 (Dataclasses, Watchdog, SQLite3)
- Database: SQLite (time-series)
- Data Format: JSON (atomic I/O)

### Running the Local Prototype

The local prototype ran as 3 concurrent scripts:

1. `Bridge.py` — connects Unreal to the data layer
2. `Harvester.py` / `Injector.py` — collects real or fabricated data
3. `WeatherMachine.uproject` — Unreal developer frontend

---

## What This Project Demonstrated

- Designing and shipping a working client-server architecture from scratch, including a live Azure deployment
- Identifying when an initial design's complexity (file-based locking, bi-directional handshakes) was solving a problem that a simpler networked architecture didn't have
- Working across a Python and C#/.NET stack, plus a C++ game engine client, in a single coherent system