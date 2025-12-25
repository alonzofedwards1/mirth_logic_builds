# mirth_logic_builds

This repository contains Mirth Connect (NextGen Connect) channel scripts for a lightweight Patient Discovery stack:

- **PD_Request_In**: HTTP Listener that accepts JSON PD trigger requests, validates required fields, and forwards a normalized payload downstream.
- **PD_Mock_Responder**: Channel Reader that builds an HL7 v3 PRPA_IN201306UV02 response and returns HTTP 202 with XML.
- **PD_Status_Get**: HTTP Listener that returns status for a given request_id.

## How to add these channels to Mirth Connect

> Tested with Mirth Connect 4.5.x using the Rhino JavaScript engine (no ES6 syntax).

### 1) Prepare the scripts
- Open `PD_channel_logic.md` and copy the **Source Transformer** JavaScript block for each channel.
- Keep the code exactly as provided; it uses only Mirth-provided APIs (logger, UUIDGenerator, DateUtil, connectorMessage, responseMap).

### 2) Create PD_Request_In (HTTP Listener)
1. New Channel → name **PD_Request_In**.
2. Source: **HTTP Listener**
   - Context Path: `/pd/trigger/` (include trailing slash so the listener matches the posted URL)
   - Allowed Methods: `POST`
   - Content-Type: `application/json`
3. Source Transformer: add a JavaScript step and paste the **PD_Request_In → Source Transformer** code. The transformer sets `responseStatusCode`/`responseHeaders`, and its return value is the HTTP response body (empty string yields an empty body).
4. Destination: **Channel Writer** (or other destination) pointing to **PD_Mock_Responder** to deliver the normalized payload.

### 3) Create PD_Mock_Responder (Channel Reader)
1. New Channel → name **PD_Mock_Responder**.
2. Source: **Channel Reader** (default settings). This receives the normalized JSON from PD_Request_In.
3. Source Transformer: add a JavaScript step and paste the **PD_Mock_Responder → Source Transformer** code. The transformer sets `responseStatusCode`/`responseHeaders`, and the returned XML string is the HTTP response body (no Response script needed).

### 4) Create PD_Status_Get (HTTP Listener)
1. New Channel → name **PD_Status_Get**.
2. Source: **HTTP Listener**
   - Context Path: `/pd/status/*`
   - Allowed Methods: `GET`
3. Source Transformer: add a JavaScript step and paste the **PD_Status_Get → Source Transformer** code. The transformer sets `responseStatusCode`/`responseHeaders`, and the returned JSON string is the HTTP response body.

### 5) Deployment notes
- Deploy all three channels after wiring the **Channel Writer** destination from PD_Request_In to PD_Mock_Responder.
- The scripts use `globalMap` keys `pd_status_<request_id>` to track request status. PD_Request_In sets `PENDING`; PD_Mock_Responder sets `COMPLETE`; PD_Status_Get reads the same key.
- Logs are written with `logger.info/warn/error` for visibility in the Channel Messages UI.
- HTTP responses come **only** from the Source Transformer return value and `responseStatusCode`/`responseHeaders` settings (Mirth 4.5.x has no separate Response script).

## How to test the channels
- See [TESTING.md](TESTING.md) for end-to-end curl commands and UI source tests for each channel.
