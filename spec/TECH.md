# cDashboard Technical Specification

## Context

cDashboard will be a new Go terminal application for the behavior specified in `spec/PRODUCT.md`. The repository currently contains only specification files, so this document defines the initial architecture for later implementation rather than describing existing code.

The product is table-heavy and read-only: a stable dashboard, service cards, service health, Plex streams, recent media, queue data, calendar data, and pending requests. The implementation should optimize for predictable rendering, testable data normalization, and safe partial-failure handling before adding richer interactions.

Primary framework choice:

1. Use `github.com/rivo/tview` for layout, tables, text views, boxes, and the application event loop.
2. Use `github.com/gdamore/tcell/v2` for terminal primitives, input events, Unicode, color, and resize handling.
3. Treat `github.com/mum4k/termdash` as a fallback only if future requirements shift toward chart-heavy widgets.

The main technical constraints come directly from `PRODUCT.md`: stable in-place refreshes, per-service error isolation, last-known-good preservation, mock mode, terminal-size gating, and no secret leakage.

## Proposed Changes

### Project Shape

Implement the app as a standard Go module with one CLI entrypoint and internal packages. The planned structure is:

```text
cDashboard/
  cmd/cdashboard/
    main.go
  internal/app/
    app.go
    state.go
    store.go
  internal/config/
    config.go
    load.go
    validate.go
  internal/clients/
    plex/
      client.go
      models.go
      sanitize.go
    sonarr/
      client.go
      models.go
      sanitize.go
    radarr/
      client.go
      models.go
      sanitize.go
    sabnzbd/
      client.go
      models.go
      sanitize.go
    overseerr/
      client.go
      models.go
      sanitize.go
  internal/httpx/
    client.go
    retry.go
  internal/logging/
    logging.go
  internal/mock/
    dashboard.go
    plex.go
    sonarr.go
    radarr.go
    sabnzbd.go
    overseerr.go
  internal/model/
    dashboard.go
    downloads.go
    media.go
    requests.go
    services.go
    streams.go
  internal/poller/
    backoff.go
    result.go
    scheduler.go
  internal/tui/
    dashboard.go
    layout.go
    panels.go
    render.go
    theme.go
    widgets.go
```

Package responsibilities:

1. `cmd/cdashboard` parses CLI flags, loads config, wires dependencies, starts logging, starts polling, and runs the TUI.
2. `internal/config` owns TOML decoding, defaults, validation, and environment-secret lookup.
3. `internal/clients` contains service-specific HTTP clients, raw API response structs, and sanitizers.
4. `internal/httpx` provides shared HTTP client setup, context timeouts, headers, JSON decoding, and safe error helpers.
5. `internal/model` contains app-owned normalized data structures. Raw API structs must not cross this boundary.
6. `internal/poller` owns refresh scheduling, manual refresh requests, service concurrency, cancellation, backoff, stale detection, and result publication.
7. `internal/app` owns dashboard state, last-known-good merge rules, and immutable snapshots for rendering.
8. `internal/tui` owns all `tview` primitives, layout, theme mapping, widget updates, keybinds, size gating, and panel row building.
9. `internal/mock` provides deterministic anonymized fixtures for mock mode and tests.
10. `internal/logging` configures file logging and redaction helpers.

### Dependency and Version Policy

Use stable, pinned Go module versions. `go.mod` and `go.sum` should be committed once code exists. Use tagged releases where possible and avoid tracking dependency `master` branches for normal development.

For TOML, prefer the simplest library that gives clear decoding and validation behavior. `github.com/pelletier/go-toml/v2` or `github.com/BurntSushi/toml` are both acceptable; choose one during implementation and keep config parsing centralized in `internal/config`.

### Runtime Model

The app has three main runtime loops:

1. The TUI event loop, owned by `tview.Application`.
2. The polling scheduler loop, owned by `internal/poller`.
3. File logging, owned by `internal/logging`.

Only the TUI event loop may mutate `tview` widgets. Background goroutines publish normalized service results or app snapshots. The dashboard applies updates through `Application.QueueUpdateDraw` to satisfy `PRODUCT.md` behavior 27 and 36.

### Data Flow

Data flows in one direction:

```text
Service APIs
  -> service clients
  -> sanitizers
  -> normalized model snapshots
  -> poller result channel
  -> app store/reducer
  -> immutable dashboard snapshot
  -> tview QueueUpdateDraw
  -> stable widgets
```

Do not let raw API response structs reach `internal/tui`. Do not let TUI widgets call service clients directly. This boundary keeps panel rendering testable and protects the UI from malformed service responses.

### State Model

Use one normalized root state in `internal/model`:

```go
type DashboardState struct {
    Services        map[ServiceName]ServiceStatus
    Plex            PlexState
    Sonarr          ArrState
    Radarr          ArrState
    SABnzbd         SABState
    Overseerr       OverseerrState
    LastUpdated     time.Time
    RefreshInterval time.Duration
}
```

Each service status should keep status metadata separate from service data:

```go
type ServiceStatus struct {
    Name        ServiceName
    Enabled     bool
    Online      bool
    Degraded    bool
    Stale       bool
    LastAttempt time.Time
    LastSuccess time.Time
    Uptime      string
    Error       string
    Backoff     time.Duration
}
```

Service poll results should represent data and failure independently so the reducer can preserve last-known-good data:

```go
type ServiceSnapshot[T any] struct {
    Name        ServiceName
    Data        T
    Err         error
    LastAttempt time.Time
    LastSuccess time.Time
    Stale       bool
}
```

Reducer rules:

1. On success, replace only that service's data, clear that service's error, set online/degraded/stale state, and update `LastSuccess`.
2. On failure, retain that service's previous data and update only its status/error fields.
3. On stale threshold breach, keep data visible and mark that service stale.
4. Never let a service update clear another service's data.
5. Never store secret values in state.

These rules implement `PRODUCT.md` behavior 28 through 35 and 48.

### Configuration

Use a human-editable TOML config with defaults applied before validation. The config shape should include:

```go
type Config struct {
    Mode            string        `toml:"mode"`
    RefreshInterval time.Duration `toml:"refresh_interval"`
    RequestTimeout  time.Duration `toml:"request_timeout"`
    StaleAfter      time.Duration `toml:"stale_after"`
    MinimumColumns  int           `toml:"minimum_columns"`
    MinimumRows     int           `toml:"minimum_rows"`
    Plex            PlexConfig    `toml:"plex"`
    Sonarr          APIConfig     `toml:"sonarr"`
    Radarr          APIConfig     `toml:"radarr"`
    SABnzbd         APIConfig     `toml:"sabnzbd"`
    Overseerr       APIConfig     `toml:"overseerr"`
}

type APIConfig struct {
    Enabled   bool   `toml:"enabled"`
    URL       string `toml:"url"`
    APIKeyEnv string `toml:"api_key_env"`
}

type PlexConfig struct {
    Enabled  bool   `toml:"enabled"`
    URL      string `toml:"url"`
    TokenEnv string `toml:"token_env"`
}
```

If the selected TOML decoder does not decode `time.Duration` from strings directly, decode duration values as strings and parse them with `time.ParseDuration` during validation.

Validation rules:

1. `mode` must be `mock` or `live`.
2. `refresh_interval` defaults to `15s` and must be positive.
3. `request_timeout` must be positive and less than `refresh_interval`.
4. `stale_after` defaults to `45s` and should be greater than `refresh_interval`.
5. `minimum_columns` defaults to `160` and must be positive.
6. `minimum_rows` defaults to `40` and must be positive.
7. In `live` mode, enabled services must have a URL and token/API-key environment variable name.
8. In `live` mode, configured secret environment variables must exist before polling starts.
9. In `mock` mode, live service URLs and credentials are not required.
10. Validation errors must name invalid keys without including secret values.

These rules implement `PRODUCT.md` behavior 38 through 43 and 48.

### HTTP Clients and Sanitization

Use one shared HTTP client configuration with per-request contexts. Service clients are responsible for authentication shape, endpoint paths, raw response structs, decoding, and conversion into normalized models.

HTTP rules:

1. Accept `context.Context` on every service call.
2. Use `context.WithTimeout` per request.
3. Set service-specific auth headers or query parameters inside the relevant client package.
4. Decode JSON into explicit structs.
5. Return errors with service name, operation, status class, and safe message.
6. Do not panic on malformed service responses.
7. Sanitize raw data before returning it to `internal/app` or `internal/tui`.
8. Redact tokens, auth headers, and private URLs from returned errors and logs.

Plex stream normalization should produce a stable `ActiveStream` model:

```go
type StreamMode string

const (
    StreamModeDirectPlay   StreamMode = "direct_play"
    StreamModeDirectStream StreamMode = "direct_stream"
    StreamModeTranscode    StreamMode = "transcode"
    StreamModeUnknown      StreamMode = "unknown"
)

type ActiveStream struct {
    User          string
    Title         string
    Device        string
    Location      string
    Quality       string
    BandwidthKbps int
    ProgressPct   int
    ETA           string
    Mode          StreamMode
    VideoCodec    string
    AudioCodec    string
    Stale         bool
}
```

Plex sanitization defaults:

1. Missing user: `unknown`.
2. Missing title: `unknown item`.
3. Missing device: `unknown device`.
4. Missing location: `unknown`.
5. Missing quality: `unknown`.
6. Missing bandwidth: `0`.
7. Invalid progress: clamp to `0..100`.
8. Unknown transcode status: `StreamModeUnknown`.
9. Missing ETA: `-`.

Sonarr and Radarr should normalize queue and calendar entries into shared app models where practical. SABnzbd should normalize download queue entries into one stable download model. Overseerr should normalize requests into one stable request model. API-specific status strings must be mapped to semantic statuses before color or warning logic is applied.

These choices implement `PRODUCT.md` behavior 13 through 23, 34, 35, and 48.

### Polling and Concurrency

Each enabled service should have a poller interface similar to:

```go
type ServicePoller interface {
    Name() model.ServiceName
    Poll(ctx context.Context) (model.ServiceData, error)
}
```

The scheduler should:

1. Run under a parent context cancelled on shutdown.
2. Tick at the configured refresh interval.
3. Accept manual refresh requests from the TUI.
4. Poll enabled services concurrently per refresh cycle.
5. Use per-service timeout contexts.
6. Keep service failures isolated and collect all results.
7. Apply per-service backoff after failures.
8. Reset backoff on success.
9. Stop all work promptly on context cancellation.
10. Publish service results without blocking the TUI event loop.

A simple `sync.WaitGroup` plus result channel is likely clearer than `errgroup` because one service failure must not cancel sibling service polling. If `errgroup` is used, it must collect service errors without violating `PRODUCT.md` behavior 28.

Default backoff policy:

1. Initial failure backoff: `5s`.
2. Maximum backoff: `2m`.
3. Small jitter to avoid synchronized retries.
4. Manual refresh may bypass current sleep but must still respect per-request timeouts.

### TUI Layout and Rendering

Use a root `Dashboard` controller with stable child widgets:

```go
type Dashboard struct {
    app       *tview.Application
    root      tview.Primitive
    sidebar   *tview.TextView
    recent    *tview.Table
    health    *tview.Table
    streams   *tview.Table
    downloads *tview.Table
    upcoming  *tview.Table
    requests  *tview.Table
    queue     *tview.Table
    footer    *tview.TextView
    theme     Theme
}
```

Build the widget tree once at startup and keep widget identities stable across refreshes. Update content in place inside one queued draw per snapshot:

```go
func (d *Dashboard) ApplySnapshot(snapshot model.DashboardState) {
    d.app.QueueUpdateDraw(func() {
        d.renderSidebar(snapshot)
        d.renderRecent(snapshot.Plex)
        d.renderHealth(snapshot)
        d.renderStreams(snapshot.Plex)
        d.renderDownloads(snapshot.SABnzbd)
        d.renderUpcoming(snapshot.Sonarr, snapshot.Radarr)
        d.renderRequests(snapshot.Overseerr)
        d.renderQueue(snapshot)
        d.renderFooter(snapshot)
    })
}
```

Panel rendering should be split into pure row-building functions and impure widget application functions:

```go
func BuildActiveStreamRows(state model.PlexState) []TableRow
func ApplyRows(table *tview.Table, rows []TableRow)
```

Pure row builders make `PRODUCT.md` behavior 11 through 24 and 36 through 37 testable without a terminal.

Initial layout should use nested `tview.Flex` containers:

```text
root vertical
  body horizontal
    sidebar fixed width
    main vertical
      top horizontal
        recently added
        health activity
      active streams
      bottom vertical
        bottom top horizontal
          downloads
          upcoming releases
        bottom bottom horizontal
          pending requests
          queue summary
  footer fixed height
```

Initial sizing:

1. Sidebar fixed width: `24` to `28` columns.
2. Footer fixed height: `1` row.
3. Top row fixed height: `10` to `12` rows.
4. Active Streams fixed height: `11` to `14` rows.
5. Bottom grid takes remaining height.
6. Gaps may use `1` column or row where needed.

Rendering rules:

1. Prepare table rows before clearing or replacing table cells.
2. Use fixed column definitions and deterministic truncation.
3. Avoid animation in the MVP.
4. Avoid frequent full layout rebuilds.
5. Batch all panel updates for a snapshot into one queued draw.
6. Render explicit empty states instead of blank panels.
7. Keep color tied to semantic status, not raw API status strings.

These rules implement `PRODUCT.md` behavior 1 through 24 and 36 through 37.

### Terminal Size Handling

Track terminal size in cells and compare it to config. The default gate is `160x40` cells.

Size behavior:

1. Check size on startup before rendering the dashboard panels.
2. Re-check size on terminal resize.
3. If below minimum, replace the app root with a size error view.
4. If size becomes valid again, restore the dashboard root.
5. Keep `R` available to retry and `Q` available to quit while the error view is displayed.

Use `tview.Application.SetBeforeDrawFunc`, a resize-aware wrapper, or explicit screen-size reads in the app loop depending on what proves least fragile during implementation. Do not implement compact mode for MVP.

These rules implement `PRODUCT.md` behavior 4 through 6.

### Styling

Define semantic colors in one theme file:

```go
type Theme struct {
    Background  tcell.Color
    PanelBorder tcell.Color
    ActiveBorder tcell.Color
    Success     tcell.Color
    Warning     tcell.Color
    Error       tcell.Color
    MutedText   tcell.Color
    PrimaryText tcell.Color
    AccentCyan  tcell.Color
    AccentGreen tcell.Color
}
```

Default palette:

| Name | Hex | Purpose |
|---|---|---|
| `Background` | `#020B17` | Main app background |
| `PanelBorder` | `#1BA6C9` | Normal panel border |
| `ActiveBorder` | `#48E5FF` | Focused panel border |
| `Success` | `#7CFF6B` | Online, healthy, complete |
| `Warning` | `#FFD166` | Stale, pending, degraded |
| `Error` | `#FF5C7A` | Offline, failed, critical |
| `MutedText` | `#7F8DA3` | Labels and low-priority values |
| `PrimaryText` | `#D6E2F0` | Main table text |
| `AccentCyan` | `#22D3EE` | Titles and cyan accents |
| `AccentGreen` | `#39FF88` | Progress and positive activity |

Unicode symbols may include status dots, progress blocks, sparkline blocks, warning/error symbols, and up/down arrows. Styling should communicate service state first and decoration second.

### Mock Mode

Mock data is a first-class input path, not a TUI-only shortcut. Store fixtures in `internal/mock` as normalized models that can feed the same app store and TUI rendering path as live service data.

Mock fixtures should include:

1. Healthy, warning, error, and stale service states.
2. Empty states for streams, downloads, releases, and requests.
3. Plex direct play, direct stream, transcode, and unknown examples.
4. Long titles and names to exercise truncation.
5. API failure snapshots that preserve last-known-good data.
6. No real hostnames, usernames, tokens, media titles from private servers, or private media-server details.

This implements `PRODUCT.md` behavior 41 and 42.

### Logging

The TUI owns stdout and stderr during normal operation. Runtime logs should go to a file once logging is configured.

Logging rules:

1. Log service name, operation, duration, and error class.
2. Redact secrets, tokens, authorization headers, and configured secret environment values.
3. Avoid logging full raw API payloads by default.
4. Do not write routine poll logs to stdout/stderr while the TUI is active.
5. Document the default log path once CLI/config behavior is implemented.

These rules implement `PRODUCT.md` behavior 35, 47, and 48.

## Testing and Validation

Automated validation should focus on packages that do not require a real terminal. The TUI should be manually checked for visual fit, flicker, and resize behavior after unit-level confidence is in place.

Unit tests:

1. Config defaults and validation cover `PRODUCT.md` behavior 25, 31, and 38 through 43.
2. Secret-env lookup and redaction cover behavior 35, 40, 43, and 48.
3. Plex stream sanitizers cover behavior 13 through 17.
4. Sonarr/Radarr/SABnzbd/Overseerr sanitizers cover behavior 18 through 23 and 35.
5. App reducer success, failure, stale, and cross-service isolation cover behavior 28 through 33.
6. Backoff and scheduler timing cover behavior 25 through 27.
7. Table row builders cover behavior 8 through 24, 36, and 37.
8. Mock fixtures are deterministic and anonymized, covering behavior 41 and 42.

Integration tests:

1. Service clients against `httptest.Server` fixtures for success, malformed payloads, HTTP errors, timeouts, and auth placement.
2. Scheduler cancellation with a parent context to prove shutdown does not hang.
3. Manual refresh path while a scheduled refresh is pending.
4. Mock-mode app startup without service credentials.
5. Live-mode config validation failure when required secret env vars are missing.

Manual verification:

1. Run mock TUI at or above `160x40` and confirm every required region from `PRODUCT.md` behavior 3 renders.
2. Resize below the minimum and back above it to confirm behavior 5 and 6.
3. Trigger service failure fixtures and confirm behavior 28 through 33.
4. Confirm normal refreshes do not visibly flash or jitter columns, covering behavior 36 and 37.
5. Confirm `Q`, `Ctrl+C`, and `R` work during normal display, during slow polling, and from the size error screen, covering behavior 26, 27, and 44.
6. Inspect logs and error screens to confirm no secrets are printed, covering behavior 48.

Expected developer commands once code exists:

```bash
go run ./cmd/cdashboard --config ./config.example.toml
go test ./...
go build ./cmd/cdashboard
```

Add `config.example.toml` when the config package exists. It must not contain real secrets.

## Risks and Mitigations

1. Plex stream and transcode fields may differ by server version, client, media type, or playback mode. Mitigate with defensive sanitizers, fixtures from multiple response shapes, and `unknown` fallbacks.
2. Some service APIs may not expose reliable uptime. Mitigate by displaying `unknown` unless the specific API behavior is verified.
3. `800x600` is pixel terminology while terminal size is measured in cells. Mitigate with configurable `minimum_columns` and `minimum_rows` defaults.
4. Excessive redraws or widget rebuilding can cause flicker. Mitigate with stable widgets and one `QueueUpdateDraw` per snapshot.
5. Table column jitter can make the dashboard hard to read. Mitigate with fixed column definitions and deterministic truncation.
6. Unicode and true color are assumed, but fonts may render symbols differently. Mitigate by keeping text labels and status words meaningful even when symbols degrade.
7. Framework choice could become wrong if the product shifts toward charts or interactive workflows. Mitigate by keeping data normalization and app state independent from `tview`.

## Follow-ups

1. Choose the TOML library during implementation and document why if the choice affects validation behavior.
2. Verify the most stable Plex endpoint for active stream bandwidth and transcode details.
3. Verify which monitored services expose reliable uptime directly.
4. Decide whether the poller should publish individual service updates as they arrive or only combined refresh-cycle snapshots after the first implementation spike.
5. Choose a default log file location for macOS, Linux, and Windows when CLI/config work starts.
6. Resolve the product open questions in `spec/PRODUCT.md` before implementing the affected panels.
