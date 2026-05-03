# cDashboard Technical Specification

## Technical Direction

cDashboard will be a Go terminal application using `tview` on top of `tcell`.

Primary framework:

- `github.com/rivo/tview`
- `github.com/gdamore/tcell/v2`

Fallback framework:

- `github.com/mum4k/termdash`

Rationale:

- `tview` provides mature tables, text views, boxes, flex layouts, grid layouts, pages, and an application event loop.
- The dashboard is table-heavy, especially the Plex active streams panel.
- `tview.Application.QueueUpdateDraw` provides a practical safe-update mechanism for background API polling.
- `tcell` provides the terminal primitives, Unicode handling, colors, input events, and resize events.
- Termdash is strong for charts, gauges, and dashboard widgets, but the MVP is denser and more table-driven than chart-driven.
- Bubble Tea/Lip Gloss remain useful comparison points, but would require more custom table/layout rendering for this version.

## Versioning Policy

Use stable, pinned Go module versions.

Guidelines:

- Use tagged releases where available.
- Do not track framework `master` branches in `go.mod` for normal development.
- Keep `go.mod` and `go.sum` committed once the codebase exists.
- Upgrade dependencies intentionally and test the TUI after upgrades.

## Runtime Model

The app has three main runtime loops:

- TUI event loop owned by `tview.Application`.
- Polling scheduler loop owned by `internal/poller`.
- Optional log writer/file output managed by `internal/logging`.

Only the TUI event loop may mutate `tview` widgets.

Background goroutines must publish normalized snapshots. The dashboard applies those snapshots through `Application.QueueUpdateDraw`.

## Directory Structure

Recommended structure:

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

- `cmd/cdashboard`: parse CLI flags, load config, wire dependencies, start app.
- `internal/config`: typed TOML config, env secret lookup, validation.
- `internal/clients`: service-specific HTTP clients and response parsing.
- `internal/model`: normalized app-owned data structures used outside client packages.
- `internal/poller`: concurrent polling, cancellation, backoff, stale detection, snapshot emission.
- `internal/app`: app state, state merging, and store/reducer logic.
- `internal/tui`: all `tview` layout, widgets, color mapping, and rendering.
- `internal/mock`: anonymized fixtures for offline mode and tests.
- `internal/logging`: file logging setup.
- `internal/httpx`: shared HTTP timeout, headers, JSON decoding, and safe error handling helpers.

## Data Flow

Data must flow in one direction:

```text
Service APIs
  -> service clients
  -> sanitizers
  -> normalized model snapshots
  -> poller result channel
  -> app state reducer/store
  -> immutable dashboard snapshot
  -> tview QueueUpdateDraw
  -> widgets
```

Do not let raw API response structs reach the TUI.

Do not let TUI widgets call service clients directly.

## State Model

Use one normalized root state.

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

Each service state should include data and status separately.

```go
type ServiceStatus struct {
    Name        ServiceName
    Enabled     bool
    Online      bool
    Stale       bool
    LastAttempt time.Time
    LastSuccess time.Time
    Uptime      string
    Error       string
    Backoff     time.Duration
}
```

Preserve last known good data independently from the latest error.

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

- On success, replace that service's data and clear its error.
- On failure, keep that service's previous data and update its error/status fields.
- On stale threshold breach, keep data visible and mark stale.
- Never let one service update clear another service's data.

## UI Model

Use a root `Dashboard` controller with stable child widgets.

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

The dashboard owns layout and widget references. Panel rendering should be split into small functions that transform normalized data into rows.

Example:

```go
func BuildActiveStreamRows(state model.PlexState) []TableRow
func ApplyRows(table *tview.Table, rows []TableRow)
```

Keep panel row builders pure so they can be tested without a terminal.

## Rendering Rules

Clean rendering is a core requirement.

Rules:

- Build the widget tree once at startup.
- Keep widget identities stable across refreshes.
- Update content in place inside `QueueUpdateDraw`.
- Never mutate widgets from polling goroutines.
- Prepare table rows before clearing/replacing table cells.
- Use fixed column definitions and deterministic truncation.
- Avoid animation in the MVP.
- Avoid frequent full layout rebuilds.
- Keep refresh interval at `15s` by default.
- Batch all panel updates for a snapshot into one queued draw.

Recommended UI update path:

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

## Terminal Size Handling

The product target is an `800x600`-class terminal window. The implementation sees terminal size as cells.

Default size gate:

- `minimum_columns = 160`
- `minimum_rows = 40`

Terminal size handling:

- On startup, check the current terminal cell size.
- On resize, re-check the size.
- If below minimum, replace the root with a size error view.
- If size becomes valid again, restore the dashboard root.
- Keep `R` available to retry and `Q` available to quit.

Implementation note:

- `tview` roots resize with the terminal.
- Use `Application.SetBeforeDrawFunc` or a lightweight resize-aware wrapper if needed.
- Do not implement compact mode for MVP unless the hard minimum later proves too strict.

## Layout Strategy

Use nested `tview.Flex` layouts first.

Wide layout:

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

- Sidebar: fixed `24-28` columns.
- Footer: fixed `1` row.
- Top row: fixed `10-12` rows.
- Active Streams: fixed `11-14` rows.
- Bottom grid: remaining height.
- Gaps: `1` column or row where needed.

Use `tview.Table` for dense panels and `tview.TextView` for service cards and footer.

## Styling

Define semantic colors in one theme package/file.

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

Unicode symbols:

- Online/status dot: `●`
- Progress filled: `█`
- Progress empty: `░`
- Sparkline blocks: `▁▂▃▄▅▆▇█`
- Warning: `!`
- Error: `x`
- Up/down: `↑` and `↓`

Avoid over-styling. Color should communicate status first.

## Configuration

Use TOML.

Recommended package options:

- `github.com/BurntSushi/toml`
- or `github.com/pelletier/go-toml/v2`

Either is acceptable. Prefer the simpler option once coding starts.

Config struct shape:

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

If the chosen TOML library does not decode `time.Duration` from strings directly, decode into string fields and parse with `time.ParseDuration` during validation.

Validation rules:

- `mode` must be `mock` or `live`.
- In `live` mode, enabled services must have a URL.
- In `live` mode, enabled services must have a token/key env name.
- Secret env vars must exist for enabled services in `live` mode.
- `refresh_interval` must be positive.
- `request_timeout` must be positive and less than `refresh_interval`.
- `stale_after` should be greater than `refresh_interval`.
- minimum columns/rows must be positive.

Never log secret values.

## HTTP Client Rules

Use one shared HTTP client configuration with per-request contexts.

Rules:

- Use `context.Context` for cancellation.
- Use `context.WithTimeout` per request.
- Set service-specific auth headers or query parameters in each client.
- Decode JSON with explicit structs.
- Return typed/normalized errors where useful.
- Do not panic on malformed service responses.
- Sanitize data before returning it to the app layer.

## Polling and Concurrency

Each enabled service should have a poller implementing:

```go
type ServicePoller interface {
    Name() model.ServiceName
    Poll(ctx context.Context) (model.ServiceData, error)
}
```

The scheduler should:

- Start one loop controlled by a parent context.
- Tick at `refresh_interval`.
- Allow manual refresh requests.
- Poll all enabled services concurrently per refresh cycle.
- Use per-service timeout contexts.
- Apply per-service backoff after failures.
- Publish a combined dashboard snapshot or individual service results to the app store.
- Stop all work promptly on context cancellation.

Preferred implementation:

- Use `errgroup` only if all service errors are collected and do not cancel sibling services unintentionally.
- A simple `sync.WaitGroup` plus result channel may be clearer for this app.

Backoff:

- Initial failure backoff: `5s`.
- Maximum backoff: `2m`.
- Reset on success.
- Add small jitter to avoid synchronized retries.

## Service Sanitization

All clients must convert raw API fields into stable internal models.

### Plex Streams

Normalize Plex active session data into:

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

Sanitization rules:

- Missing user: `unknown`
- Missing title: `unknown item`
- Missing device: `unknown device`
- Missing quality: `unknown`
- Missing location: `unknown`
- Missing bandwidth: `0`
- Invalid progress: clamp to `0..100`
- Unknown transcode status: `StreamModeUnknown`
- Missing ETA: `-`

### Other Services

Sonarr and Radarr should normalize queue and calendar entries into shared app models where practical.

SABnzbd should normalize download queue entries into one stable download model.

Overseerr should normalize requests into one stable request model.

Do not expose API-specific status strings directly to color/style logic. Map them to semantic statuses first.

## Mock Data

Mock data is a first-class input path.

Requirements:

- Deterministic default dataset.
- No real hostnames, usernames, tokens, or media-server identifiers.
- Enough rows to exercise truncation and dense rendering.
- Include healthy, warning, error, and stale states.
- Include Plex direct play, direct stream, transcode, and unknown examples.
- Include API failure snapshots while preserving last known good data.

Tests should be able to import mock fixtures without starting the TUI.

## Logging

The TUI owns stdout and stderr during normal operation.

Logging rules:

- Log to a file, not stdout.
- Default log path should be documented once CLI/config exists.
- Redact secrets and auth headers.
- Include service name, operation, duration, and error class.
- Avoid logging full raw API payloads by default.

## Testing Strategy

Test without relying on terminal rendering wherever possible.

Unit tests:

- Config loading and validation.
- Secret-env lookup behavior with redaction.
- API response sanitizers.
- Poller success/failure state transitions.
- Stale data detection.
- Backoff behavior.
- Table row builders.
- String truncation and progress bar formatting.

Integration tests:

- API clients against local `httptest.Server` fixtures.
- Scheduler cancellation with context timeout.
- Mock mode app startup without service credentials.

Manual verification:

- Run mock TUI at valid size.
- Resize below minimum and back above minimum.
- Simulate API failure and confirm last good data stays visible.
- Confirm no obvious flicker during refresh.

## Build and Run Targets

Initial expected commands once code exists:

```bash
go run ./cmd/cdashboard --config ./config.example.toml
go test ./...
go build ./cmd/cdashboard
```

Add a sample config file once the config package exists:

```text
config.example.toml
```

Do not include real secrets in examples.

## Risks

- Plex stream and transcode fields may differ by server version, client, media type, or playback mode.
- Some service APIs may not expose reliable uptime.
- `800x600` is pixel terminology, while the TUI receives character-cell dimensions.
- Excessive redraws or widget rebuilding can cause flicker.
- Table column jitter can make the dashboard hard to read.
- Unicode and true color are assumed, but user terminal fonts may still render some symbols poorly.
- Termdash may become more attractive if charting becomes more important than tables.
- Bubble Tea may become more attractive if the product shifts from dashboard monitoring to interactive workflows.

## Technical Open Questions

- Which TOML library should be selected after a quick implementation spike?
- Which Plex endpoint provides the most stable stream bandwidth and transcode details for the current stable Plex release?
- Which services expose uptime directly, and which should display `unknown`?
- Should service polling publish individual service updates as they arrive or only publish a combined refresh-cycle snapshot?
- What default log file location should be used on Windows, Linux, and macOS?
