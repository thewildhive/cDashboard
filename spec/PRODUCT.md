# cDashboard Product Specification

## Purpose

cDashboard is a Go-based terminal dashboard for monitoring a home media server stack from one dense, glanceable view.

The dashboard monitors:

- Plex
- Sonarr
- Radarr
- SABnzbd
- Seerr

The product should favor clear operational visibility over decorative animation. The target user is technical enough to configure local services and API keys, but should not need to maintain a complex custom-rendered terminal application.

## Product Goals

- Show the health and activity of the media stack in one terminal screen.
- Preserve known good data when one service fails or times out.
- Make API failures visible without destroying the rest of the dashboard.
- Keep the UI readable, stable, and scannable during refreshes.
- Provide an offline/mock mode for development, demos, and tests.
- Keep the MVP small enough to build and validate service by service.

## Non-Goals

- This is not a full replacement for Plex, Sonarr, Radarr, SABnzbd, or Seerr web UIs.
- The MVP will not edit queues, approve requests, delete downloads, or mutate server state.
- The MVP will not support tiny terminal layouts.
- The MVP will not prioritize animations, mouse-first interaction, or theme customization.
- The MVP will not attempt pixel-perfect reproduction of the mockup when terminal constraints make that fragile.

## Display Requirements

The hard minimum product target is an `800x600`-class terminal window.

Terminal applications usually receive terminal size as character cells, not pixels. The implementation must therefore enforce a configured minimum grid size. The default grid threshold should be chosen to preserve the full dashboard layout on common modern terminal fonts, and it must be easy to adjust in config.

Initial implementation defaults:

- `minimum_columns = 160`
- `minimum_rows = 40`
- Target comfortable size: `180x45` cells or larger
- If the terminal is too small, render only a clear size error screen

Small-terminal behavior:

```text
cDashboard requires a larger terminal.
Minimum: 160x40 cells, 800x600-class window.
Current: 132x32 cells.

Resize the terminal and press R to retry, or Q to quit.
```

Unicode is acceptable. The product assumes a modern terminal with Unicode and color support.

## Dashboard Layout

The visual target is a dark blue/cyan/neon-green terminal dashboard inspired by the supplied mockup.

Required regions:

- Left sidebar with one service card per monitored service.
- Top row with `Recently Added` and `Health & Activity` panels.
- Main center `Active Streams` table for Plex sessions.
- Bottom grid with `Downloads`, `Upcoming Releases`, `Pending Requests`, and `Queue Summary`.
- Footer with keybinds, refresh interval, current view, and last updated timestamp.

### Sidebar

Each service card should show:

- Service name
- Status dot
- Online/offline/degraded/stale state
- Uptime when available from the service API
- API health/error state
- A compact activity sparkline or text-only activity indicator
- Service-specific summary metrics

Examples:

- Plex: active streams, users, bandwidth
- Sonarr: queue count, grabbed count, RSS count
- Radarr: queue count, grabbed count, RSS count
- SABnzbd: queue count, speed, queue size
- Seerr: total requests, pending requests, issue count

### Top Row

`Recently Added` should show recently added Plex items.

`Health & Activity` should show per-service status and lightweight resource/activity metrics where available.

### Active Streams

The `Active Streams` panel is the most important table in the app.

It should show:

- User
- Item title
- Location
- Device
- Quality
- Bandwidth
- Playback mode, such as direct play, direct stream, transcode, or unknown
- Progress bar
- ETA

Plex stream/transcode data must be sanitized before rendering so incomplete or inconsistent API responses cannot corrupt the UI.

### Bottom Grid

`Downloads` should show active and queued SABnzbd jobs.

`Upcoming Releases` should show Sonarr/Radarr calendar data.

`Pending Requests` should show Seerr requests.

`Queue Summary` should aggregate queue counts and status across services.

### Footer

The footer should be one row tall and include:

- App name and version
- Keybinds
- Current view/focus
- Refresh interval
- Last updated timestamp
- Global stale/error indicator when relevant

## Core Behavior

### Refreshing

- The refresh interval is configurable.
- Default refresh interval should be `15s`.
- Manual refresh should be available with `R`.
- Background polling must never block keyboard input or screen redraw.

### API Failures

API failures are handled per service.

On success:

- Replace that service's data with the latest sanitized snapshot.
- Clear the service error state.
- Update `last_success`.

On failure:

- Keep the last known good data.
- Store the new error message for that service.
- Update `last_attempt`.
- Show the error in the service card and health panel.
- Mark the data stale once it exceeds the stale threshold.

Failure of one service must not blank the dashboard or block other services.

### Stale Data

Each service should track freshness independently.

Default stale threshold:

- `stale_after = 45s`

Stale data should remain visible but clearly marked.

### Uptime

Use service APIs for uptime where possible.

If a service does not expose reliable uptime, show `unknown`. Do not infer uptime from unrelated fields unless the API behavior has been verified.

### Mock Mode

The app must support anonymized fake data for:

- UI development without real services
- Non-connected demo mode
- Automated tests
- Golden/snapshot-style panel rendering tests

Mock data must not contain real usernames, tokens, hostnames, or private media-server details.

## Configuration

Use a human-editable TOML config file with environment-variable references for secrets.

Requirements:

- Configurable service URLs
- Configurable API key/token environment variable names
- Configurable refresh interval
- Configurable request timeout
- Configurable stale threshold
- Configurable minimum terminal size
- `mock` and `live` modes
- Validation with clear error messages

Secrets should be read from environment variables, not stored directly in config files.

Example:

```toml
mode = "live"
refresh_interval = "15s"
request_timeout = "5s"
stale_after = "45s"
minimum_columns = 160
minimum_rows = 40

[plex]
enabled = true
url = "http://plex.local:32400"
token_env = "CDASHBOARD_PLEX_TOKEN"

[sonarr]
enabled = true
url = "http://sonarr.local:8989"
api_key_env = "CDASHBOARD_SONARR_API_KEY"

[radarr]
enabled = true
url = "http://radarr.local:7878"
api_key_env = "CDASHBOARD_RADARR_API_KEY"

[sabnzbd]
enabled = true
url = "http://sabnzbd.local:8080"
api_key_env = "CDASHBOARD_SABNZBD_API_KEY"

[Seerr]
enabled = true
url = "http://Seerr.local:5055"
api_key_env = "CDASHBOARD_Seerr_API_KEY"
```

## Keybinds

MVP keybinds:

- `Q` or `Ctrl+C`: quit
- `R`: refresh now
- `Tab`: cycle focused panel, if focus is implemented
- Arrow keys: scroll focused table, if scrolling is implemented

Mouse support is deferred.

## MVP Scope

The smallest useful first version includes:

- Runnable TUI layout using fake data
- Hard minimum terminal-size gate
- TOML config loading and validation
- Mock/offline mode
- Real service health checks
- Plex active streams
- Plex recently added
- SABnzbd queue summary and active downloads
- Sonarr/Radarr calendar and queue summaries
- Seerr pending requests
- Concurrent polling with timeouts
- Per-service error and stale states
- Last known good data preservation
- File logging

## Deferred Scope

Defer until the dashboard is stable:

- Mutating actions such as approving requests or deleting queue items
- Mouse support
- Historical charts
- Notification integrations
- Multi-theme support
- Plugin system
- Prometheus/exporter mode
- Advanced compact layouts for small terminals
- Complex animations

## Development Milestones

1. Build the static TUI layout with deterministic mock data.
2. Add the minimum terminal-size gate.
3. Add typed config loading and validation.
4. Add normalized internal data models.
5. Add mock data fixtures and non-connected mode.
6. Add service API clients one at a time.
7. Add concurrent polling with context cancellation, timeouts, and backoff.
8. Wire snapshots into the UI safely.
9. Add stale/error rendering and last-known-good behavior.
10. Add tests for config, sanitization, state merging, and panel row generation.

## Acceptance Criteria

- The app starts in mock mode with no external services configured.
- The app refuses to render the full dashboard below the configured minimum terminal size.
- The dashboard does not visibly flash or rebuild the whole screen on normal refresh.
- Failure of one service does not clear other services or crash the app.
- Failure of one service does not clear that service's last known good data.
- Stale data is visibly marked per service.
- All service API polling is cancellable on shutdown.
- API tokens are not printed to stdout, logs, or error screens.
- The core panel row-building logic can be tested without a terminal.

## Open Product Questions

- Which Plex account/user naming convention should be displayed when multiple names are available?
- Should local and remote Plex streams be grouped or only tagged by location?
- Which queue states from Sonarr/Radarr/SABnzbd deserve warning colors?
- Should Seerr approved-but-not-fulfilled requests appear in `Pending Requests` or a future separate panel?
