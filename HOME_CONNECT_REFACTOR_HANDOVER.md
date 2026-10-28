# Home Connect recovery refactor handover

Prepared 28 October 2026 (Europe/London).

## Workspace and status

- Fork: https://github.com/stuartbean/home-assistant-core
- Working branch: `fix/home-connect-recovery`
- Base: upstream Home Assistant Core `2026.10.4`, commit `357cae5c598dfcd7a0ea7dc5b1facccd4ec6037b`.
- This matches the user's installed HA version. The fork's default branch remains dev.
- Only this handover has been added. No integration implementation has been changed or deployed.
- The original investigation was explicitly read-only. The user subsequently authorized adding this handover to GitHub. Do not interpret this note as authorization to change the live HA installation.
- Develop and test in a separate WSL2 checkout on its Linux filesystem. Inspect repository AGENTS.md and development instructions first. Later assess/rebase or port onto current upstream dev before proposing an upstream PR.
- Keep this planning document out of an eventual upstream code PR unless appropriate.

Suggested checkout:

```sh
git clone https://github.com/stuartbean/home-assistant-core.git
cd home-assistant-core
git switch fix/home-connect-recovery
git remote add upstream https://github.com/home-assistant/core.git
```

## User's objective and working scenario

Improve handling of intermittent Wi-Fi for a powered-on Bosch washing machine, possibly idle/standby and producing only connectivity events. Keep appliance issues isolated from other appliances. Replace crude refresh suppression with controlled recovery and a useful connection-quality diagnostic.

Working assumptions for this exercise: HA and washer share the local network; the HC stream is healthy; repeated appliance CONNECTED/DISCONNECTED events result from intermittent washer Wi-Fi. Do not keep redirecting the investigation to speculative internet/cloud failures. Still model API failures correctly.

The user accepts HA's documented unavailable convention once connectivity is established through API verification. No separate entity grace period is required by the agreed design. Do not assume losing a reading means erasing the cached value.

## Verified current behaviour (2026.10.4)

Primary files: homeassistant/components/home_connect/{coordinator.py,entity.py,binary_sensor.py,sensor.py,__init__.py,diagnostics.py}, plus tests/components/home_connect/.

- aiohomeconnect==0.39.1.
- One shared SSE stream: GET https://api.home-connect.com/api/homeappliances/events, via Client.stream_all_events().
- Each appliance has a separate HomeConnectApplianceCoordinator, data cache, entity listeners and _execution_tracker.
- The outer dispatcher routes by event_message.ha_id. For an existing appliance, except DEPAIRED, it changes connected from false to true and notifies all that appliance's entity listeners BEFORE interpreting the event.
- DISCONNECTED then sets that appliance's data.info.connected=false and notifies its entity listeners. No API call, timer, recovery loop, reconnect-counter increment or integration reload is started by that branch.
- Therefore another DISCONNECTED while already disconnected can publish unavailable -> cached values (where entity-specific checks permit) -> unavailable.
- Normal entity availability uses appliance.info.connected and _attr_available, with additional checks in some subclasses.
- CONNECTED/PAIRED check a per-appliance rolling count: append every qualifying event; at >=8 events within 3600 seconds skip the full refresh. Suppressed events still count. No preceding disconnect is required.
- Warning is emitted on crossing into suppression according to the existing list-length condition; warning occurrences are not a count of reconnect events.
- Live STATUS/NOTIFY/EVENT/DISCONNECTED processing continues. The outer connected=true logic happens before suppression.
- No timer checks expiry or performs a catch-up refresh. Another qualifying event must arrive when the rolling count is <8 (including itself).
- The shared stream reader awaits appliance.event_listener(); that handler can await full API refresh and rate-limit sleep, delaying consumption for ALL appliances.
- Full refresh obtains appliance info and writes connected from its response. Any HomeConnectError from that info request also writes connected=false, conflating failure to verify with confirmed device disconnection.
- API-confirmed disconnected causes get_appliance_data() to raise UpdateFailed. This currently also makes last_update_success false even though an API verification succeeded.
- Ordinary refresh errors do not create a periodic appliance retry loop. TooManyRequestsError in full refresh sleeps for Retry-After/default and retries.
- Shared SSE transport interruption has its own backoff: 10s, doubling up to 3600s; a delivered event resets it. Distinct from appliance recovery.
- Existing HomeConnectConnectivityBinarySensor is diagnostic, reports appliance.info.connected, and overrides availability to coordinator.last_update_success. This is a useful extension point but its availability conflates API failure and API-confirmed disconnected.
- Entity creation/listeners have special paired/depaired lifecycle behaviour. Preserve discovery, program-option updates, authentication, entity identities and unload semantics.

### Startup and API reads

Setup GET /api/homeappliances discovers appliances, starts SSE and schedules a background initial full refresh per appliance.

A washer full refresh uses GET requests (base https://api.home-connect.com):
1. /api/homeappliances/{haId}: appliance metadata and connected.
2. /api/homeappliances/{haId}/settings: settings/power state.
3. /api/homeappliances/{haId}/status: grouped current status.
4. /api/homeappliances/{haId}/programs: program list and selected/active program info.
5. /api/homeappliances/{haId}/programs/available/{programKey}: option definitions when applicable.
6. /api/homeappliances/{haId}/commands: available commands.

If appliance info reports disconnected, remaining grouped reads are skipped. Some entity-specific metadata fetches can occur additionally. Not all event-derived values have a REST snapshot equivalent.
Refresh GETs do not control the physical appliance, but current cache mutations/notifications are not side-effect-free. Settings/status are merged while some other collections are cleared/replaced; inspect exact merge semantics before changing.

### KEEP-ALIVE clarification

KEEP-ALIVE is an APPLICATION-LEVEL SSE heartbeat for the event stream, not an appliance heartbeat and not HTTP keep-alive. Documented interval is 55 seconds; client read timeout 60 seconds. aiohomeconnect filters these out before yielding to HA.
SSE parsers retain last-event-id across messages, so a heartbeat can expose the previous appliance ID without proving that appliance connected. A maintainer explicitly warned about this. No raw wire capture established whether HC also repeats an explicit id field.

## Agreed design direction

### Separate facts and recovery

- .connected is authoritative API-reported appliance connectivity; event handlers must not overwrite it. "Read-only" means read-only to event consumers, not an immutable API model.
- A failed API request preserves the last confirmed .connected value. API reachability/verification health is separate.
- CONNECTED/DISCONNECTED nudge a per-appliance verification/recovery process, rather than directly changing connectivity or triggering repeated unbounded full refreshes.
- Successful appliance-info response with connected=false is a successful connectivity check, not itself an API failure.
- Retry periodically while recovery is active (about five minutes was suggested, not fixed). Coalesce nudges, bound minimum spacing, respect Retry-After and API request budgets.
- Recovery can be active while the last confirmed .connected is still true.
- On confirmed recovery fetch/reconcile full data without blocking the shared SSE reader.
- Remove eight-events/hour as a mechanism disabling updates. Instability can inform diagnostics.
- Lifecycle tasks/timers must cancel on unload/depair and avoid overlaps.

### One queued state-application path

Both live events and API results enter a normalized message queue; the dispatcher becomes the consumer and the sole writer of appliance state. API workers fetch, normalize and submit results rather than directly mutating cached state.

Preserve message provenance and semantics:
- Status/settings messages update relevant properties.
- API appliance-info messages alone update authoritative .connected.
- Connection events nudge recovery and feed instability assessment.
- API errors affect verification health, not .connected.
- Preserve special paired/depaired and program/entity-discovery handling.

Keep SSE RUNNING during refresh. Stopping SSE during snapshots was discussed and superseded because of missed-event gaps and shared-stream impact.

### Critical concurrency rule: live events take precedence

Use per-appliance optimistic revision validation (user accepts best implementation; monotonic revision preferred over wall-clock timestamps):
1. Capture the appliance's ingress revision before starting snapshot calls.
2. Fetch the data without mutating live models/caches.
3. Normalize into one snapshot batch.
4. Atomically compare the current appliance revision with the captured revision and enqueue the ENTIRE batch only if unchanged.
5. Otherwise reject the entire batch and schedule a bounded retry.

Any accepted live event for that appliance advances its ingress revision WHEN ENQUEUED, not when consumed. Other appliances do not invalidate the snapshot. Validation + insertion must be atomic relative to live enqueue. Already queued updates are accounted for. Batch application/notification must avoid intermediate partial snapshot states. Do not pretend queue FIFO alone prevents a late stale API response overwriting a newer live event.

Rejected batches must not modify .connected, API-health flags or caches inadvertently. Decide explicit handling of lifecycle changes/depair and failed fetches. Prevent starvation/request storms under frequent updates; respect the user's conservative whole-device rejection rule rather than silently substituting field-level merging.

### Connectivity quality diagnostic

Add an enum diagnostic sensor (prefer keeping existing binary connectivity entity for compatibility):
- GOOD: API-confirmed connected, stable.
- POOR: API-confirmed connected, recent connectivity instability.
- NONE: API-confirmed disconnected.
- HA unavailable: API cannot be reached/verification cannot be established.

"none" must be a string enum value, not Python None (which produces unknown).
NONE takes precedence over instability when confirmed disconnected. Exact POOR thresholds and recovery hysteresis remain to be specified. Quality should improve after instability ages out even with no new device events.
Call this Home Connect connection quality; do not imply measured Wi-Fi RSSI.
API unavailability and appliance disconnection must be distinguishable. A single request error is not necessarily an integration-wide outage; classify errors/auth/rate limiting appropriately. A loaded HA config entry has no entity-like unavailable state.

## History and rationale

The connected=true-on-event logic was introduced 4 March 2025 by PR #139760 ("Fix home connect available"). Its test demonstrates an API refresh failure followed by a STATUS event restoring all entities for that appliance. Thus event-based recovery was intentional, but no justification was found for applying it to DISCONNECTED.
Review discussion rejected marking all appliances disconnected just because the shared SSE stream broke: idle appliances may produce no events indefinitely.
Do not characterize all maintainers' competence; identify concrete state/concurrency defects.

Sources:
- Current coordinator: https://github.com/home-assistant/core/blob/2026.10.4/homeassistant/components/home_connect/coordinator.py
- Entity base: https://github.com/home-assistant/core/blob/2026.10.4/homeassistant/components/home_connect/entity.py
- Connectivity sensor: https://github.com/home-assistant/core/blob/2026.10.4/homeassistant/components/home_connect/binary_sensor.py
- Startup: https://github.com/home-assistant/core/blob/2026.10.4/homeassistant/components/home_connect/__init__.py
- PR: https://github.com/home-assistant/core/pull/139760
- Introducing commit: https://github.com/home-assistant/core/commit/47033e587bc843cabba3f44a060e6e22a477cf66
- Stream/idle discussion: https://github.com/home-assistant/core/pull/139760#discussion_r1979676666
- Keep-alive warning: https://github.com/home-assistant/core/pull/139760#discussion_r1979789654
- Original issue: https://github.com/home-assistant/core/issues/139654
- Client source: https://aiohomeconnect.readthedocs.io/en/latest/_modules/aiohomeconnect/client.html
- HC API: https://api-docs.home-connect.com/
- HC events: https://api-docs.home-connect.com/events
- HA availability: https://contributors.home-assistant.io/core/quality-scale/entity-unavailable/

## Validation priorities

Run repository-required checks and relevant Home Connect tests in supported Linux/Python environment. Add meaningful regression tests for:
- No connected=true bounce on repeated DISCONNECTED.
- Only successful appliance-info responses write connected; API failure preserves last confirmation.
- Confirmed disconnected => quality NONE, diagnostic available, normal entities unavailable.
- API failure => diagnostic unavailable; distinguish auth/rate limiting from device disconnection.
- Event arriving during API fetch rejects whole batch with no partial mutation.
- Other-device events do not reject the batch.
- Pending queue events, atomic compare/enqueue, and events after accepted batches preserve ordering.
- Shared SSE continues while one appliance fetches/sleeps/retries.
- Repeated nudges coalesce; periodic recovery occurs without new events; no overlapping workers.
- POOR never disables updates, and can age back to GOOD without more events.
- Startup disconnected, reconnection, partial failures, full refresh/entity discovery, selected/active program semantics.
- Shutdown/depair cancels tasks/timers and prevents stale in-flight results being applied.
- Existing entity unique IDs/binary connectivity and normal platforms remain compatible.

No fork has been cloned locally and no tests have been run yet. The next agent should read this note, inspect repository instructions, establish the WSL development environment, and agree/execute the implementation scope with the user.
