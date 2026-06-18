# KNX ESP32 Driver (Berry) – Idea and Functionality

## Overview

This driver connects a Tasmota-based ESP32 device (with SCD41 sensor telemetry) to a KNX IoT backend API.
Its main purpose is to:

- read sensor values from `Tele#SCD41`
- map those values to configured KNX group addresses (GA)
- resolve GAs to backend datapoint UUIDs at boot
- write changed values to the backend in bulk (`write_many`)

The runtime is intentionally lightweight:

- no KNX callback subscriptions
- no continuous GA discovery
- UUID mapping is prepared at startup and cached

## Core Idea

The design follows a boot-resolve-run pattern:

1. Boot checks network and API availability.
2. OAuth tokens are obtained for read and write scopes.
3. Configured KNX group addresses are resolved once to backend UUIDs.
4. Mapping is cached in persistent storage.
5. During runtime, incoming telemetry values are compared with last sent values.
6. Only changed values are sent to the backend in a single bulk request.

This reduces traffic, simplifies runtime behavior, and improves reliability on embedded hardware.

## State Machine

The driver uses explicit states for visibility and debugging:

- `INIT`
- `NETWORK_WAIT`
- `API_CHECK`
- `AUTH_INIT`
- `CACHE_LOAD`
- `CACHE_VALIDATE`
- `GA_RESOLVE`
- `READY`
- `RUN`
- `DEGRADED`

If a critical startup step fails, it transitions to `DEGRADED`.

## Data Flow

1. Tasmota publishes SCD41 telemetry (`Tele#SCD41`).
2. `on_scd41_update(...)` reads fields:
   - `CarbonDioxide`
   - `Temperature`
   - `Humidity`
   - `DewPoint`
3. Each field is matched to a configured GA (`tele_key` mapping).
4. GA is translated to UUID using the runtime map.
5. Values are change-filtered using `_last_value_dump`.
6. Changed items are sent via `PUT /api/v2/datapoints/values`.

## Configuration

Important constants in `knx_driver.be`:

- `CFG_API_URL` - backend base URL
- `CFG_API_BASE` - API prefix (default `/api/v2`)
- `CFG_OAUTH_ID` / `CFG_OAUTH_SECRET` - OAuth client credentials
- `CFG_CACHE_VERSION` - cache schema/version guard
- `CFG_POINTS` - list of KNX targets (GA, DPT, telemetry key, critical flag)

Example point entry:

```json
{"ga":"3/1/21", "dpt":"9.001", "tele_key":"Temperature", "critical":true}
```

## OAuth and API Usage

The driver uses client-credentials OAuth:

- `POST /oauth/access` with scope `read` for GA lookup
- `POST /oauth/access` with scope `write` for datapoint writes

Runtime API calls:

- health check: `GET /health`
- GA lookup: `GET /api/v2/datapoints?filter[ga]=...`
- value write: `PUT /api/v2/datapoints/values`

On `401`, the driver refreshes the token and retries once.

## Caching Strategy

The resolved GA-to-UUID mapping is stored in persistent memory:

- key: `persist.knx_driver_cache`
- structure:
  - `version`
  - `map` (`ga -> { uuid, dpt, tele_key, critical }`)

Cache is accepted only if:

- version matches `CFG_CACHE_VERSION`
- all critical GAs exist

If invalid, the driver resolves mappings from the API.

## Change Detection and Bulk Write

To avoid unnecessary traffic:

- each field value is serialized (`json.dump`)
- compared with the last sent serialized value per GA
- only changed fields are added to the bulk payload

Bulk payload format:

```json
{
  "data": [
    { "id": "<uuid>", "attributes": { "value": "<stringified-value>" } }
  ]
}
```

## Reliability Features

- network wait loop before startup logic
- API health check before OAuth and resolve
- retry logic for GA resolution with backoff
- token refresh on unauthorized responses
- degraded mode for critical startup failures
- explicit state transition logs for observability

## Limitations / Current Scope

- no inbound KNX callback handling
- no dynamic runtime subscription system
- fixed telemetry field set for SCD41
- values are sent as stringified payload attributes

## Startup and Lifecycle

- Global singleton `knx_driver` is created.
- `knx_start()` is called automatically at a file load.
- `knx_stop()` currently logs stop state only (no teardown required in this mode).

## Typical Use Case

This driver is ideal when you want:

- a simple ESP32 edge component
- deterministic startup mapping
- minimal runtime complexity
- efficient periodic updates from local sensors into a KNX IoT semantic backend
