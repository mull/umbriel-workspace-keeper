# umbriel-workspace-keeper

When a monitor is unplugged, Umbriel moves all of its windows onto the
*active* workspace of a remaining output. This daemon moves each one to the
workspace with the same name or position there instead (`web` → `web`,
`2` → `2`, `3` → `3`), and moves them back when the monitor returns.

Upstream request for doing this in the compositor:
[noctalia-dev/umbriel#302](https://github.com/noctalia-dev/umbriel/issues/302).

## How it works

- Listens to `umbriel subscribe windows,workspaces` and remembers each
  window's home (output and workspace).
- When a window shows up on another output because its home output went away,
  it is moved to the matching workspace with `window-focus:<id>` +
  `window-move-to-workspace-silent:<ws>/<output>`, and remembered as parked.
  Keyboard focus is restored afterwards.
- When the home output comes back, parked windows are moved back to it.
- Umbriel evacuates windows in its own internal order, not layout order, so
  afterwards each workspace's old left-to-right order is rebuilt with
  `column-move-to-last`. Column actions apply to the output under the pointer,
  so this step warps the pointer to each window it reorders.
- Windows that were already on the matching workspace aren't moved, only
  reordered, so Umbriel's own restore brings them back with their exact layout.
- State lives in `$XDG_STATE_HOME/umbriel-workspace-keeper/state.json`, so a
  daemon restart while undocked doesn't lose track.

Moving a window explicitly clears Umbriel's built-in memory of its old home,
which is why the move back is handled here too.

## Requirements

- Umbriel with the `subscribe` IPC command (tested with 0.1.0, `eed6c60ba450`)
- Python 3.10 or newer, standard library only
- A systemd user session, for the included unit

## Install

From a clone of this repository:

```sh
ln -sf "$PWD/umbriel-workspace-keeper" ~/.local/bin/
ln -sf "$PWD/umbriel-workspace-keeper.service" ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now umbriel-workspace-keeper
journalctl --user -u umbriel-workspace-keeper -f
```

The unit starts with `umbriel-session.target`, which Umbriel's packaged
systemd session provides. Without it, start the daemon from Umbriel's config
instead:

```toml
[general]
autostart = ["umbriel-workspace-keeper"]
```

Run `umbriel-workspace-keeper --dry-run -v` to log moves without making them.

## Different monitors in different places

Windows normally wait for the monitor they came from, identified by connector
name such as `DP-2`. If you undock at home and dock at work, the work monitor
may show up as another connector, and the windows would stay on the laptop.

With `--adopt-new-outputs`, when exactly one external monitor is missing and
a different external monitor appears, the windows waiting for the missing one
move to the new one instead. Built-in panels (`eDP`, `LVDS`, `DSI`) and
virtual outputs never take part. With more than one monitor missing or
appearing at once, windows only return to their own monitor.

To turn it on for the systemd unit, add an override with
`systemctl --user edit umbriel-workspace-keeper`:

```ini
[Service]
ExecStart=
ExecStart=%h/.local/bin/umbriel-workspace-keeper --adopt-new-outputs
```

## Testing without unplugging

A virtual output behaves like a monitor when it is destroyed:

```sh
umbriel output-create test 1280x720
# put some windows on test's workspaces
umbriel output-destroy test   # windows mirror onto the remaining output
umbriel output-create test 1280x720   # and move back
```

## Limitations

- Windows briefly appear on one workspace before being spread out.
- Windows stacked in one column, or in a tab group, come back as separate
  columns when the daemon moved them. Left-to-right order is kept.
- Scratchpad windows are left to Umbriel.
- A monitor that disconnects when it sleeps triggers the same shuffle (and
  the move back on wake).
