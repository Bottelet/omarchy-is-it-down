# Status Watcher

*Is it me or is it down?*

An [Omarchy](https://omarchy.org) bar plugin that watches the status pages of
services you depend on — GitHub, AWS, Cloudflare, npm and more — from a little
detective icon in the bar.

![Status Watcher panel](preview.png)

- The detective sits quietly in the bar while everything is operational.
- When a watched service reports trouble, the icon tints yellow/red and shows
  a count badge (how many services are affected).
- Click it to open a panel with one tab per service. Each tab is bordered and
  dotted green / yellow / red by that service's state; the detail card shows
  the overall status, active incidents, and per-component health.
- Middle-click a tab (or use the "Open … status page" link) to open the real
  status page in your browser.
- Middle-click the bar icon to force a refresh.

Everything follows the active Omarchy theme: text, fonts and the "red"
(outage) color come from the theme; green/yellow ship with sensible defaults
you can override (see below).

## Install

```bash
omarchy plugin add https://github.com/Bottelet/omarchy-status-watcher.git --enable
omarchy bar put bottelet.status-watcher --after omarchy.weather
```

Or manually:

```bash
git clone https://github.com/Bottelet/omarchy-status-watcher.git \
  ~/.config/omarchy/plugins/bottelet.status-watcher
omarchy-shell shell rescanPlugins
omarchy plugin enable bottelet.status-watcher
omarchy bar put bottelet.status-watcher --after omarchy.weather
```

## Configure

**In the panel (recommended):** click the ⚙ gear in the panel header.

- **Service list page** — every service with an ENABLED/DISABLED toggle.
  Click a service row to drill into it.
- **Per-service page** — AWS shows its full region list; statuspage services
  (GitHub, Cloudflare, …) show their components. Each row has its own
  ENABLED/DISABLED toggle. A filter field narrows big lists (Cloudflare has
  ~470 PoPs), and **Enable all / Disable all** buttons apply to the current
  filter matches — e.g. filter "china", Disable matches.
- **Mute from the status view** — hover any component/event row and click the
  ✕ that appears; it lands in the same per-service ignore list.

Muted entries don't badge the icon and don't show in the panel. When ignore
rules are active for a statuspage service, its severity is recomputed from
the remaining components (the site-wide indicator is only trusted while real
incidents are open), so muting the noise can turn a service green.

All of it persists to the widget's inline entry in
`~/.config/omarchy/shell.json` and applies immediately.

**Via CLI:** settings hot-apply the same way. The manifest also declares a
settings `schema` (services + AWS regions multi-selects) for when Omarchy
ships a widget-settings form.

**Comma caveat**: quickshell's IPC CLI splits arguments on commas, so
`omarchy bar set ... '["a","b"]' --json` fails for multi-value lists. Either
pass the list as a single comma-separated string (supported for regions):

```bash
omarchy bar set bottelet.status-watcher awsIgnoreRegions "me-central-1,me-south-1"
```

…or edit `shell.json` directly for proper JSON arrays (applies live):

```bash
jq '(.bar.layout[] | .[] | select(.id == "bottelet.status-watcher")).services = ["github","aws","claude"]' \
  ~/.config/omarchy/shell.json > /tmp/shell.json && mv /tmp/shell.json ~/.config/omarchy/shell.json
```

Single-value settings work fine through the CLI:

```bash
# Watch a single extra service list entry / simple values
omarchy bar set bottelet.status-watcher services '["github"]' --json

# Poll every 10 minutes (default: 3)
omarchy bar set bottelet.status-watcher refreshMinutes 10

# Override status colors (red defaults to the theme's urgent color)
omarchy bar set bottelet.status-watcher okColor '#a6e3a1'
omarchy bar set bottelet.status-watcher warnColor '#f9e2af'
omarchy bar set bottelet.status-watcher downColor '#f38ba8'
```

An empty `services` selection means the default set: GitHub, AWS, Cloudflare,
npm.

### AWS regions

AWS events from regions you don't care about can be ignored — they won't
badge the icon or appear in the panel. Use the **AWS: ignore regions**
multi-select in the settings UI, or the CLI (region codes or display names
both work, comma-separated):

```bash
omarchy bar set bottelet.status-watcher awsIgnoreRegions "me-central-1,me-south-1"
# clear again (watch all regions):
omarchy bar set bottelet.status-watcher awsIgnoreRegions ""
```

## Adding a service

Any Atlassian Statuspage-powered site (most developer services) works out of
the box. Add an entry to `registry()` in `Model.js`:

```js
{ key: "tailscale", name: "Tailscale", type: "statuspage", defaultEnabled: false,
  api: "https://status.tailscale.com/api/v2/summary.json", page: "https://status.tailscale.com" },
```

…and mirror it in `manifest.json` under `barWidget.schema[0].options` so it
appears in the settings form. Then rescan:

```bash
omarchy-shell shell rescanPlugins
```

Tip: to check whether a service uses Statuspage, try
`curl https://<status-host>/api/v2/status.json`.

## Remove

```bash
omarchy plugin remove bottelet.status-watcher
```

This deletes the plugin folder and drops it from the bar. To also clear its
saved settings, remove the `bottelet.status-watcher` entry from
`~/.config/omarchy/shell.json`. The plugin never touches any other
configuration.

## Dependencies

Only tools present on a stock Omarchy install: `curl` for fetching status
APIs and `iconv` (glibc) for the UTF-16 AWS feed. No extra packages, no
background processes beyond the shared Omarchy shell.

## Notes

- Fetch failures are shown as "Unreachable — maybe it's you" and deliberately
  do **not** light up the detective: if the status pages themselves are
  unreachable, the problem is probably your own connection.
- AWS has no Statuspage; the plugin reads the public AWS Health current-events
  feed instead (`type: "aws"`).
- IPC: `omarchy-shell ipc call bottelet.status-watcher toggle|open|close|refresh`

## License

MIT
