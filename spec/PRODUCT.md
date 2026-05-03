# cDashboard Product Specification

## Summary

cDashboard is a Go-based terminal dashboard for monitoring a home media server stack from one dense, glanceable view. It gives a technical home-server operator read-only visibility into Plex, Sonarr, Radarr, SABnzbd, and Overseerr without requiring them to keep several browser tabs open or maintain a complex custom terminal UI.

## Problem

The monitored services expose useful health, queue, playback, and request data, but each service presents that data in a separate UI. cDashboard should make the combined operational state visible in one stable terminal screen while keeping partial failures understandable instead of letting one bad service degrade the whole dashboard.

## Goals

1. Show the health and activity of the media stack in one terminal screen.
2. Keep the dashboard readable, stable, and scannable during refreshes.
3. Preserve known-good data when one service fails or times out.
4. Make API failures and stale data visible per service without blanking unrelated data.
5. Support an offline/mock mode for development, demos, and automated tests.
6. Keep the first implementation small enough to validate service by service.

## Non-goals

1. cDashboard is not a replacement for the Plex, Sonarr, Radarr, SABnzbd, or Overseerr web UIs.
2. The MVP does not approve requests, delete downloads, edit queues, or mutate server state.
3. The MVP does not support tiny terminal layouts or compact alternate dashboards.
4. The MVP does not prioritize animations, mouse-first interaction, or user-defined themes.
5. The MVP does not need pixel-perfect reproduction of any visual mockup when terminal constraints would make that fragile.

## Behavior

1. When launched at or above the configured minimum terminal size, cDashboard renders a full-screen terminal dashboard using a dark blue/cyan/neon-green visual style. The interface favors dense operational visibility over decorative motion.

2. The dashboard monitors the following services when they are enabled in configuration: Plex, Sonarr, Radarr, SABnzbd, and Overseerr. A disabled service remains visible only if the implementation chooses to show disabled cards; if shown, it must be clearly marked disabled and must not appear failed.

3. The main dashboard layout contains these regions in the default view: a left service sidebar, a top row with `Recently Added` and `Health & Activity`, a central `Active Streams` table, a bottom grid with `Downloads`, `Upcoming Releases`, `Pending Requests`, and `Queue Summary`, and a one-row footer.

4. The dashboard targets an `800x600`-class terminal window. Because terminal apps receive size in character cells, the product default minimum is `160` columns by `40` rows, with a comfortable target of `180x45` cells or larger.

5. If the current terminal is below the configured minimum size, cDashboard does not render the dashboard panels. It renders a clear size error screen that includes the required size, the current size, and the available actions to retry or quit.

6. From the size error screen, pressing `R` retries the size check and pressing `Q` quits. If the terminal becomes large enough, the app returns to the full dashboard without requiring a restart.

7. The app assumes a modern terminal with Unicode and color support. Unicode symbols may be used for status dots, progress bars, sparklines, and directional indicators, but the dashboard must remain understandable when symbols render imperfectly.

8. Each service card in the sidebar shows the service name, status, freshness, API health, and service-specific summary metrics. A service card must distinguish at least `online`, `offline`, `degraded`, `stale`, and `disabled` when those states apply.

9. Plex summary metrics include active streams, users, and bandwidth when available. Sonarr and Radarr summary metrics include queue count, grabbed count, and RSS or calendar activity when available. SABnzbd summary metrics include queue count, speed, and queue size when available. Overseerr summary metrics include total requests, pending requests, and issue count when available.

10. The sidebar may show a compact sparkline or text-only activity indicator per service. The absence of sparkline data must not be treated as an error; the card may show a stable placeholder such as `unknown` or `-`.

11. `Recently Added` shows recent Plex items in a compact list or table. Missing optional metadata such as year, library, poster, or added timestamp must not prevent the item title from rendering.

12. `Health & Activity` shows per-service status and lightweight resource or activity metrics where available. This panel must make current errors and stale states visible without requiring the user to inspect logs.

13. `Active Streams` is the primary table in the app. It shows active Plex sessions with user, item title, location, device, quality, bandwidth, playback mode, progress, and ETA when those values are available.

14. Plex stream and transcode data must be sanitized before rendering. Missing or inconsistent stream fields render as safe placeholders rather than corrupting alignment, creating blank rows, or crashing the app.

15. Active stream playback mode renders as `direct play`, `direct stream`, `transcode`, or `unknown`. Unknown or inconsistent Plex transcode indicators must render as `unknown`, not as an inferred value.

16. Active stream progress renders as a bounded value from `0` through `100`. Invalid, negative, missing, or over-maximum progress data must be clamped or replaced with a safe placeholder before display.

17. If there are no active Plex streams, `Active Streams` renders an explicit empty state such as `No active streams`, not a blank panel.

18. `Downloads` shows active and queued SABnzbd jobs. Each visible job should show enough information to understand what is downloading, progress, speed, size or remaining size, and status when available.

19. If SABnzbd has no active or queued jobs, `Downloads` renders an explicit empty state such as `No active downloads`.

20. `Upcoming Releases` shows relevant Sonarr and Radarr calendar entries. Entries must identify whether they come from Sonarr or Radarr when both services are enabled.

21. If there are no upcoming releases in the configured lookahead window, `Upcoming Releases` renders an explicit empty state.

22. `Pending Requests` shows Overseerr requests that still require attention. Approved-but-not-fulfilled requests are product-ambiguous and should remain an open question until implementation confirms the desired grouping.

23. `Queue Summary` aggregates queue counts and status across Sonarr, Radarr, SABnzbd, and any other enabled service with queue-like state. The panel must not double-count the same item across services unless the data source really reports distinct queue entries.

24. The footer is one row tall and includes the app name, version when available, keybind hints, current view or focus when focus is implemented, refresh interval, last updated timestamp, and a global stale/error indicator when any service needs attention.

25. The default refresh interval is `15s`. The interval is configurable and must be shown in the footer or equivalent status area.

26. Pressing `R` triggers a manual refresh. Manual refresh must remain available while the dashboard is otherwise idle, while a previous refresh is finishing, and from the size error screen.

27. Background polling must not block keyboard input, quitting, terminal resize handling, or redraws. The user must be able to press `Q` or `Ctrl+C` to quit even when service calls are slow or failing.

28. Polling failures are handled per service. A failure from one service must not clear, hide, or mark unrelated services as failed.

29. On a successful service refresh, cDashboard replaces that service's visible data with the latest sanitized snapshot, clears that service's current error, records `last_success`, and updates the displayed freshness state.

30. On a failed service refresh, cDashboard keeps that service's last known good data visible, records the new error message for that service, records `last_attempt`, and marks the service degraded or offline as appropriate.

31. A service's data becomes stale when its last successful refresh exceeds the configured stale threshold. The default stale threshold is `45s`.

32. Stale data remains visible but is clearly marked stale in the service card and any relevant panel. Stale marking must be per service, not global-only.

33. If a service has never successfully loaded, its panels or rows show an unavailable/error state rather than pretending stale data exists.

34. Service uptime is shown when a service API exposes reliable uptime. If reliable uptime is unavailable or unverified, cDashboard shows `unknown` rather than inferring uptime from unrelated fields.

35. API failures must be visible in the service card and `Health & Activity`. Error text should be concise enough to fit the dashboard and must not expose secrets, tokens, or full private URLs.

36. The dashboard updates in place during normal refreshes. It should not visibly flash, rebuild the whole screen, reorder stable rows unnecessarily, or jitter table columns when values of similar shape update.

37. Long text values such as titles, usernames, device names, or request names are truncated deterministically when they do not fit. Truncation must preserve table alignment and should keep the most useful identifying text visible.

38. Config is provided through a human-editable TOML file. Secrets are referenced by environment variable name in config and are read from the environment at runtime.

39. Config supports at least service URLs, service token or API-key environment variable names, refresh interval, request timeout, stale threshold, minimum terminal size, and `mock` versus `live` mode.

40. Config validation failures are shown as clear startup errors. Validation errors must identify the invalid setting and expected shape without printing secret values.

41. In `mock` mode, cDashboard starts without real service URLs or credentials and renders deterministic anonymized data that exercises healthy, warning, error, empty, and stale states.

42. Mock data must not contain real usernames, tokens, hostnames, media titles from the user's private server, or other private media-server details.

43. In `live` mode, enabled services require valid connection settings and a configured environment variable name for their token or API key. Missing live-mode secrets must fail validation before polling starts.

44. MVP keybinds are `Q` or `Ctrl+C` to quit and `R` to refresh. If panel focus is implemented, `Tab` cycles focus and arrow keys scroll the focused table.

45. Mouse support is deferred. The dashboard must remain fully usable for MVP monitoring with the keyboard-only controls above.

46. The MVP is read-only. No displayed action may imply that cDashboard can mutate queues, approvals, downloads, libraries, or service configuration.

47. File logging is allowed, but normal dashboard operation owns the terminal screen. Logs must not write routine diagnostic output into the dashboard's stdout/stderr stream.

48. API tokens, authorization headers, and secret values must never appear in dashboard panels, startup errors, stdout/stderr, or logs.

49. The smallest useful first version includes static mock-mode layout, terminal-size gating, config loading and validation, real service health checks, Plex active streams, Plex recently added, SABnzbd queue data, Sonarr/Radarr calendar and queue summaries, Overseerr pending requests, concurrent polling with timeouts, per-service stale/error states, last-known-good preservation, and file logging.

50. Deferred scope includes mutating actions, mouse support, historical charts, notifications, multi-theme support, plugin support, Prometheus/exporter mode, advanced compact layouts, and complex animations.

51. **Open question:** Which Plex account or user naming field should be displayed when multiple names are available?

52. **Open question:** Should local and remote Plex streams be grouped separately, or only tagged by location in the stream row?

53. **Open question:** Which queue states from Sonarr, Radarr, and SABnzbd deserve warning colors rather than neutral styling?

54. **Open question:** Should Overseerr approved-but-not-fulfilled requests appear in `Pending Requests` or in a future separate panel?
