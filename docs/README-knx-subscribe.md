# KNX Subscription Driver (Berry) – Idea and Functionality

## Overview

This driver enables a **Tasmota-based ESP32 device** to **subscribe to KNX CoV (Change of Value) notifications** from a KNX IoT backend API.
Its main purpose is to:

- establish HTTP callback subscriptions to specific KNX datapoints
- listen for incoming Change-of-Value events from the backend
- display or process incoming notifications in real-time
- automatically renew subscriptions before they expire
- gracefully handle inactivity and cleanup

The runtime is event-driven and lightweight:

- no local sensor polling
- no outbound writes (receive-only)
- callback-based notification handling
- automatic subscription lifecycle management

---

## Core Idea

The design follows a **subscribe-listen-renew** pattern:

1. **OAuth authentication** fetches manage and read tokens.
2. **Datapoint lookup** resolves a configured KNX group address to its backend UUID.
3. **HTTP callback subscription** is created (backend posts notifications to Tasmota's IP).
4. **Tasmota httpserver** registers the callback endpoint and begins listening.
5. **Incoming notifications** are parsed, formatted, and displayed.
6. **Subscription lifetime** is monitored and renewed at half-life to prevent expiration.
7. **Inactivity timeout** optionally shuts down the listener if no events arrive within a threshold.

This enables real-time push notifications from the KNX backend to edge devices.

---

## State Machine

The driver uses modular startup and cleanup phases:

- `INIT` → network and OAuth checks
- `RESOLVE` → GA-to-UUID lookup
- `LISTEN` → HTTP server starts, callback registered
- `SUBSCRIBE` → subscription created on backend
- `RUNNING` → awaiting notifications
- `RENEW` → subscription lifetime extended
- `CLEANUP` → subscription deleted, server stopped

On fatal errors, the startup sequence aborts and logs the failure reason.

---

## Data Flow

### Subscription Flow (Boot)

1. `start_knx_subscription()` is called.
2. OAuth tokens (`manage`, `read`) are fetched via `POST /oauth/access`.
3. Configured GA (e.g., `1/1/114`) is resolved to datapoint metadata via `GET /api/v2/datapoints?filter[ga]=...`.
4. Tasmota's built-in httpserver is started.
5. Callback endpoint is registered (e.g., `POST http://192.168.1.109:8080/knx_cov`).
6. Subscription is created on backend via `POST /api/v2/subscriptions` with the datapoint UUID and callback URL.
7. Subscription ID and lifetime are captured.

### Notification Flow (Runtime)

1. Backend detects a CoV event on the subscribed datapoint.
2. Backend POSTs JSON payload to the Tasmota callback URL.
3. `handle_notification()` receives and parses the payload.
4. Fields are extracted: `ga`, `value`, `timestamp`, `dpt`.
5. Formatted message is printed to the console.

### Renewal Flow

1. Every half-subscription-lifetime, `renew_subscription()` is called.
2. Subscription is extended via `PATCH /api/v2/subscriptions/{id}` with a fresh lifetime.
3. A new renewal timer is scheduled.

### Cleanup Flow

1. On restart, restart or manual `stop_knx_subscription()`:
2. Subscription is deleted via `DELETE /api/v2/subscriptions/{id}`.
3. HTTP server is stopped.

---

## Configuration

Important constants in `knx_subscribe.be`:

- `CFG_API_URL` - backend base URL
- `CFG_API_BASE` - API prefix (default `/api/v2`)
- `CFG_OAUTH_ID` / `CFG_OAUTH_SECRET` - OAuth client credentials
- `CFG_GROUP_ADDRESS` - KNX GA to subscribe to (e.g., `1/1/114`)
- `CFG_CALLBACK_HOST` - IP address of this Tasmota instance (must be reachable from backend)
- `CFG_CALLBACK_PORT` - HTTP listener port (typically `8080`)
- `CFG_CALLBACK_PATH` - endpoint path (default `/knx_cov`)
- `CFG_LIFETIME_MIN` - subscription lifetime in minutes
- `CFG_INACTIVITY_S` - auto-shutdown after X seconds without notification

Example configuration:

```berry
var CFG_API_URL       = "http://knx-runtime-engine.example.org"
var CFG_GROUP_ADDRESS = "1/1/114"
var CFG_CALLBACK_HOST = "192.168.1.109"
var CFG_LIFETIME_MIN  = 5
var CFG_INACTIVITY_S  = 60
```

---

## OAuth and API Usage

The driver uses client-credentials OAuth:

- `POST /oauth/access` with scope `manage` for subscription lifecycle (create, renew, delete)
- `POST /oauth/access` with scope `read` for datapoint lookup

Runtime API calls:

- GA lookup: `GET /api/v2/datapoints?filter[ga]=...`
- create subscription: `POST /api/v2/subscriptions`
- renew subscription: `PATCH /api/v2/subscriptions/{id}`
- delete subscription: `DELETE /api/v2/subscriptions/{id}`

On `401`, the driver logs a warning but does not auto-retry (subscriptions are long-lived).

---

## Subscription Lifecycle

### Creation

Subscriptions are created with:
- **subscription type**: `callback`
- **callback URL**: `http://CFG_CALLBACK_HOST:CFG_CALLBACK_PORT/CFG_CALLBACK_PATH`
- **lifetime**: configurable (default 5 minutes)
- **datapoints**: array containing the target datapoint UUID

Example payload:

```json
{
  "data": {
    "type": "subscription",
    "attributes": {
      "subscriptionType": "callback",
      "url": "http://192.168.1.109:8080/knx_cov",
      "lifetime": {"minutes": 5}
    },
    "relationships": {
      "subscriptionDatapoints": {
        "data": [{"type": "datapoint", "id": "<uuid>"}]
      }
    }
  }
}
```

### Renewal

At half the subscription lifetime (e.g., 2.5 minutes for a 5-minute subscription):

```json
{
  "data": {
    "type": "subscription",
    "id": "<subscription-id>",
    "attributes": {"lifetime": {"minutes": 5}}
  }
}
```

### Deletion

On shutdown or restart:

```
DELETE /api/v2/subscriptions/{subscription-id}
```

---

## Notification Format

Incoming notifications are expected in this format:

```json
{
  "data": [
    {
      "id": "<datapoint-uuid>",
      "type": "datapoint",
      "meta": {
        "ga": "1/1/114",
        "datapointId": "...",
        "dpt": "9.001"
      },
      "attributes": {
        "title": "Temperature Room A",
        "value": 21.5,
        "timestamp": "2026-06-18T14:23:45Z"
      }
    }
  ]
}
```

The driver extracts and logs:
- GA
- datapoint name (title)
- value (stringified via `json.dump`)
- DPT (if available)
- timestamp (if available)

---

## Inactivity Monitoring

The driver can automatically shut down if no notifications arrive within a configurable window:

- checks every 5 seconds since the last event
- if elapsed time exceeds `CFG_INACTIVITY_S`, terminates gracefully
- useful for temporary monitoring or test scenarios

Set `CFG_INACTIVITY_S = 0` to disable inactivity check (recommended for production).

---

## Error Handling

The driver handles:

- **network unavailability** - fails at startup, logs reason
- **OAuth failures** - logs HTTP code and response
- **invalid GA** - fails if GA cannot be resolved, logs GA
- **subscription creation failures** - logs HTTP code, aborts
- **malformed notifications** - logs raw payload, continues listening
- **renewal failures** - logs warning but continues (subscription valid until expiration)

---

## Reliability Features

- explicit startup phase logging
- timestamp formatting for all events
- HTTP status code validation on every API call
- graceful cleanup on shutdown
- inactivity detection for autonomous operation
- automatic renewal before subscription expiration
- detailed console output for debugging

---

## Limitations / Current Scope

- receive-only (no outbound writes from this driver)
- single datapoint subscription per instance
- no built-in persistence of notifications
- callback must be reachable from backend (no firewall/NAT bypass)
- HTTP callbacks are unencrypted (no TLS in Tasmota httpserver by default)

---

## Startup and Lifecycle

- Global variables and helper functions are defined.
- `start_knx_subscription()` is called automatically at a file load.
- Manual stop: `stop_knx_subscription()`.
- Subscription is deleted on restart or explicit stop.

---

## Typical Use Cases

This driver is ideal when you want:

- a simple ESP32 listener for KNX datapoint changes
- real-time push notifications (not polling)
- minimal runtime overhead
- integration with Tasmota for sensor/actor management
- bridge between KNX semantic backend and local automation logic

---

## Example Output

```
── Starting KNX Subscription ─────────────────────────
[14:22:10] ↺ OAuth 'manage' succeeded
[14:22:11] ↺ OAuth 'read' succeeded
Callback URL : http://192.168.1.109:8080/knx_cov
Datapoint    : GA=1/1/114  datapointId=room-a-temp  UUID=550e8400-e29b-41d4-a716-446655440000  "Temperature Room A"

[14:22:12] ✓ Subscription created: sub-uuid-12345
Waiting for notifications... (stop_knx_subscription() to exit)

[14:23:45] → CoV received (187 bytes)
[14:23:45] ★ Update  GA=1/1/114  "Temperature Room A"  value=21.5  dpt=9.001  ts=2026-06-18T14:23:45Z

[14:24:50] → CoV received (189 bytes)
[14:24:50] ★ Update  GA=1/1/114  "Temperature Room A"  value=21.7  dpt=9.001  ts=2026-06-18T14:24:50Z

[14:24:32] ↺ Subscription renewed (lifetime 5 min)
```

---

## Integration with Other Drivers

- **knx_driver.be** (outbound writes) + **knx_subscribe.be** (inbound notifications) form a bidirectional KNX IoT bridge
- Both can run independently or together on the same ESP32
- They share OAuth credentials and API URLs but operate on different datapoints
