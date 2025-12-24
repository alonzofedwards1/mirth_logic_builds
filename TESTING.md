# Testing the Patient Discovery Channels

These steps assume all three channels are deployed in Mirth Connect 4.5.x and that the HTTP Listeners are bound to `http://localhost:8080`.
Adjust host/port if your listener differs.

## 1) PD_Request_In (HTTP Listener)
- **Happy path:**
  - Command:
    ```bash
    curl -i -X POST http://localhost:8080/pd/trigger/ \
      -H "Content-Type: application/json" \
      -d '{"request_id":"REQ-123","patient_id":"PAT-456"}'
    ```
  - Expected:
    - HTTP `202 Accepted`
    - Empty response body (Source Transformer returns empty string)
    - Channel log shows `Accepted PD trigger requestId=REQ-123 patientId=PAT-456`
    - Global status key `pd_status_REQ-123` set to `PENDING`
- **Empty body:**
  - Command: `curl -i -X POST http://localhost:8080/pd/trigger/ -H "Content-Type: application/json" -d ''`
  - Expected: HTTP `400 Bad Request` with body `Empty request body`
- **Invalid JSON:**
  - Command: `curl -i -X POST http://localhost:8080/pd/trigger/ -H "Content-Type: application/json" -d '{'`
  - Expected: HTTP `400 Bad Request` with body `Invalid JSON payload`
- **Missing fields:**
  - Command: `curl -i -X POST http://localhost:8080/pd/trigger/ -H "Content-Type: application/json" -d '{"request_id":"REQ-123"}'`
  - Expected: HTTP `400 Bad Request` with body `Missing required fields: request_id and patient_id`

## 2) PD_Mock_Responder (Channel Reader)
- **End-to-end via PD_Request_In:**
  - First, send a happy-path trigger as above.
  - The Channel Writer destination from `PD_Request_In` should hand off the normalized JSON to `PD_Mock_Responder`.
  - Expected in `PD_Mock_Responder` message view:
    - `responseStatusCode` set to `202`
    - `responseHeaders` contains `Content-Type: application/xml`
    - Returned HL7 v3 PRPA_IN201306UV02 XML visible as the message content (no Response script is used)
    - Global status key `pd_status_<request_id>` updated to `COMPLETE`
- **Direct source test (UI):**
  - In the Dashboard, select `PD_Mock_Responder` → **Send Message** (Channel Reader Source Test).
  - Paste a normalized JSON payload: `{ "request_id": "REQ-789", "patient_id": "PAT-999" }` and send.
  - Expected: message content is HL7 v3 XML, status code `202`, and `pd_status_REQ-789` becomes `COMPLETE`.

## 3) PD_Status_Get (HTTP Listener)
- **Status found:**
  - After sending a trigger (e.g., `REQ-123`), run:
    ```bash
    curl -i http://localhost:8080/pd/status/REQ-123
    ```
  - Expected:
    - HTTP `200 OK`
    - JSON body with `request_id`, `status` (`PENDING` or `COMPLETE`), and `timestamp`
- **Status via query parameter:**
  - Command: `curl -i "http://localhost:8080/pd/status/?request_id=REQ-123"`
  - Expected: same as above (`200 OK` with JSON body)
- **Status not found:**
  - Command: `curl -i http://localhost:8080/pd/status/UNKNOWN-ID`
  - Expected: HTTP `404 Not Found` with JSON body `{ "error": "request_id not found" }`
- **Missing request_id segment:**
  - Command: `curl -i http://localhost:8080/pd/status/`
  - Expected: HTTP `404 Not Found` with JSON body `{ "error": "request_id not found" }`

## 4) Channel log verification
- Use the **Dashboard → Messages** view to confirm log entries for accepted/rejected triggers and XML responses.
- Filter by channel name to see lifecycle logs (`logger.info/warn/error`).

## 5) Resetting test state
- To clear cached statuses, redeploy the channels or stop/start the Mirth server. Status keys are stored in `globalMap` and are not persisted across redeploys or service restarts.
