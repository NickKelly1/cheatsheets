# cmus Cheatsheet

cmus is a lightweight, keyboard-driven terminal music player. Its interface is
organized into seven views; every action is a command that can also be typed
after `:`, bound to a key, placed in a configuration file, or sent remotely.

Last checked: 2026-07-21. Examples target cmus 2.12.

## Table of Contents

- [Five-minute start](#five-minute-start)
- [Install and inspect](#install-and-inspect)
- [Views](#views)
- [Playback and modes](#playback-and-modes)
- [Navigate, search, and select](#navigate-search-and-select)
- [Build the library, playlists, and queue](#build-the-library-playlists-and-queue)
- [Command mode](#command-mode)
- [Configuration and key bindings](#configuration-and-key-bindings)
- [Remote control](#remote-control)
- [Troubleshooting](#troubleshooting)
- [Wide quick reference](#wide-quick-reference)
- [References](#references)

---

## Five-minute start

```bash
cmus
```

Inside cmus, add a directory, open the library, and play a track:

```text
:add ~/Music        Add ~/Music recursively to the library
1                   Open the tree library
j / k               Move down / up
Space               Expand the selected artist or album
Tab                 Switch between the tree and track panes
Enter               Play the selected track
c                   Pause or resume
q                   Quit
```

The library and settings are saved automatically. Removing an entry from a
cmus view does not normally delete the audio file; deleting a file from browser
view `5` does, after confirmation.

---

## Install and inspect

| Debian / Ubuntu | Fedora | Arch Linux | macOS (Homebrew) |
| --- | --- | --- | --- |
| `sudo apt install cmus` | `sudo dnf install cmus` | `sudo pacman -S cmus` | `brew install cmus` |

```bash
cmus --version        # version and build features
cmus --plugins        # available input, output, and effect plugins
cmus --show-cursor    # visible cursor, useful for screen readers
man cmus
man cmus-remote
```

The audio formats and output systems available depend on how the installed
package was built. Check `cmus --plugins` when a format or device is missing.

---

## Views

The number keys are global shortcuts. Each view changes what selection,
`Space`, `a`, `y`, `e`, `E`, and `D` act on.

| Key | View | Layout / purpose | Typical action |
| ---: | --- | --- | --- |
| `1` | Library | Artist/album tree plus tracks | `Space` expands; `Tab` changes pane |
| `2` | Sorted library | Flat, sortable library | Filter or browse every track |
| `3` | Playlists | Playlist panel plus tracks | `Space` marks the active playlist |
| `4` | Play queue | Tracks that play before the normal source | Reorder or remove upcoming tracks |
| `5` | Browser | Filesystem browser | `Enter` opens; `Backspace` goes up |
| `6` | Filters | Saved library filters | `Space` selects; `Enter` activates |
| `7` | Settings | Bindings, commands, and options | `Enter` edits; `Space` toggles |

`Tab` moves between panes where a view has more than one. Press `i` in views
1–3 to jump to the currently playing track.

---

## Playback and modes

| Key | Playback | Key | Playback | Key | Mode toggle | Key | Mode toggle |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `x` | Play/restart track | `c` | Pause/resume | `r` | Repeat | `s` | Shuffle |
| `v` | Stop | `b` | Next track | `C` | Continue | `M` | Library/playlist source |
| `z` | Previous track | `B` | Next album | `m` | All/artist/album | `o` | Tree/sorted library |
| `Z` | Previous album | `Ctrl-R` | Repeat current | `t` | Remaining/elapsed time | `f` | Follow playing track |
| `+` / `-` | Volume +/-10% | `[` / `{` | Left channel +/-1% | `]` / `}` | Right channel +/-1% | | |
| `h` / `l` | Seek -/+5 seconds | `Left` / `Right` | Seek -/+5 seconds | `,` / `.` | Seek -/+1 minute | | |

The status line shows active modes. `continue` controls whether playback
continues after the current track; `repeat` repeats the current source;
`shuffle` changes its order. `m` changes the library scope among all tracks,
the current artist, and the current album.

Exact bindings can vary by package or personal configuration. View them in
view `7`, or query one with `:showbind common c`.

---

## Navigate, search, and select

| Key | Move | Key | Move | Key | Find / select | Key | Window |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `j` / `k` | Down / up | `g` / `G` | Top / bottom | `/text` | Search forward | `Tab` | Next pane |
| `Ctrl-F/B` | Page down / up | `Ctrl-D/U` | Half-page down / up | `?text` | Search backward | `Space` | Expand, mark, or toggle |
| `Ctrl-E/Y` | Scroll down / up | `Home` / `End` | Top / bottom | `n` / `N` | Next / previous match | `Enter` | Activate or play |
| | | | | `//text` | Artist/album search in view 1 | `i` | Current playing track |

Search normally matches artist, album, and title in views 1–4. If a track has
no tags, cmus compares the filename. Press `Esc` or `Ctrl-C` to cancel search
or command input.

In flat list views 2–4, `Space` marks tracks. Editing commands operate on all
marked tracks, or on the cursor item when nothing is marked.

---

## Build the library, playlists, and queue

### Add from any view

| Destination | Command | From views 1–5 | Meaning |
| --- | --- | --- | --- |
| Library | `:add -l ~/Music` | `a` | Add selected files/directories to library |
| Marked playlist | `:add -p FILE` | `y` | Copy selection to marked playlist |
| End of queue | `:add -q FILE` | `e` | Play selection before normal source |
| Front of queue | `:add -Q FILE` | `E` | Make selection next in queue |

Paths may be files, directories, supported URLs, or playlists. Quote paths
containing spaces: `:add "/mnt/music/My Albums"`.

### Edit and persist

| Task | Key / command | Task | Key / command |
| --- | --- | --- | --- |
| Remove selected/marked entries | `D` or `Delete` | Clear current view | `:clear` |
| Move after cursor | `p` | Move before cursor | `P` |
| Unmark everything | `:unmark` | Invert marks | `:invert` |
| Create playlist | `:pl-create road-trip` | Rename selected playlist | `:pl-rename new-name` |
| Import playlist | `:pl-import list.m3u` | Export selected playlist | `:pl-export list.m3u` |
| Save current view | `:save list.m3u` | Save queue explicitly | `:save -q queue.m3u` |
| Load into library views | `:load -l list.m3u` | Update library metadata | `u` |

The queue has priority. After it empties, cmus resumes the library or playlist
that was playing. Moving tracks with `p`/`P` is disabled when that view is
automatically sorted.

---

## Command mode

Press `:` to enter command mode, `Tab` to complete, `Up`/`Down` for history,
and `Enter` to run. Command names may be abbreviated only when unambiguous.

| Goal | Command | Goal | Command |
| --- | --- | --- | --- |
| Show an option | `:set shuffle?` | Set an option | `:set shuffle=true` |
| Toggle an option | `:toggle shuffle` | Set volume | `:vol 65%` |
| Relative volume | `:vol +5%` | Relative seek | `:seek +30` |
| Absolute seek | `:seek 1:30` | Temporary filter | `:filter artist="Björk"` |
| Live filter | `:live-filter beatles` | Remove filter | `:factivate` |
| Change theme | `:colorscheme green` | Redraw display | `:refresh` |
| Rescan changed files | `:update-cache` | Force full rescan | `:update-cache -f` |
| Show working directory | `:pwd` | Change directory | `:cd ~/Music` |
| Quit | `:quit` or `:wq` | Help entry point | `:help` |

Useful playback commands include `:player-play`, `:player-pause`,
`:player-next`, `:player-prev`, and `:player-stop`.

---

## Configuration and key bindings

User files normally live under `$XDG_CONFIG_HOME/cmus`, defaulting to
`~/.config/cmus`:

| File | Purpose | File | Purpose |
| --- | --- | --- | --- |
| `rc` | Hand-written commands loaded at startup | `autosave` | State cmus saves automatically |
| `cache` | Library metadata cache | `*.theme` | User color schemes |

Example `~/.config/cmus/rc`:

```text
set softvol=true
set softvol_state=70 70
set replaygain=track
set continue=true
set status_display_program=

bind -f common space player-pause
bind -f common n player-next
bind -f common p player-prev
bind -f common q quit -i
```

Use `-f` for personal bindings so an existing system or autosaved binding can
be replaced. Binding contexts are `common`, `library`, `playlist`, `queue`,
`browser`, and `filters`.

```text
:bind -f common F2 toggle shuffle
:unbind -f common F2
:showbind common F2
:source ~/.config/cmus/rc
```

Test commands interactively before putting them in `rc`. View `7` is the
easiest place to discover option names and current bindings.

---

## Remote control

`cmus-remote` talks to a running cmus instance through its local Unix socket.

| Play | Pause | Stop | Previous / next |
| --- | --- | --- | --- |
| `cmus-remote -p` | `cmus-remote -u` | `cmus-remote -s` | `cmus-remote -r` / `cmus-remote -n` |

| Volume / seek | Modes | Status | Raw cmus command |
| --- | --- | --- | --- |
| `cmus-remote -v +5%` | `cmus-remote -R` repeat | `cmus-remote -Q` | `cmus-remote -C 'toggle continue'` |
| `cmus-remote -k +30` | `cmus-remote -S` shuffle | `cmus-remote -C status` | `cmus-remote -C 'set replaygain=track'` |

Add media remotely:

```bash
cmus-remote -l "$HOME/Music"          # add to library
cmus-remote -q '/path/to/song.flac'   # add to queue
cmus-remote -c playlist.m3u           # clear and load playlist view
```

Extract a few status fields:

```bash
cmus-remote -Q | awk '
  /^status / { state=$2 }
  /^tag artist / { sub(/^tag artist /, ""); artist=$0 }
  /^tag title / { sub(/^tag title /, ""); title=$0 }
  END { printf "%s: %s — %s\n", state, artist, title }
'
```

Keep remote control on the default Unix socket. `cmus --listen host:port` is
insecure even with a password and must never be exposed to the internet. Do
not use alternate sockets to run multiple instances as the same user; they can
corrupt the shared metadata cache.

---

## Troubleshooting

| Symptom | Check | Likely fix |
| --- | --- | --- |
| No sound | `:set output_plugin?`; `cmus --plugins` | Select a built output plugin and check the system mixer |
| Volume keys do nothing | `:set softvol?`; check hardware mixer | `:set softvol=true`, then `:vol 70%` |
| File will not play | `cmus --plugins`; inspect the file/container | Install a build with the required input codec |
| New/changed tags absent | Select tracks and press `u` | Run `:update-cache`; use `-f` only for a full rescan |
| Deleted files remain | Press `u` in library | Run `:update-cache` to remove missing entries |
| Remote says connection refused | Confirm cmus is running as same user | Start cmus; check `$CMUS_SOCKET` / runtime directory |
| Binding is ignored | `:showbind CONTEXT KEY` | Use the right context and `bind -f` |
| Terminal display is damaged | `Ctrl-L` | Resize terminal or run `:refresh` |
| Playlist move does nothing | Check `pl_sort` / `lib_sort` in view `7` | Disable automatic sorting before manual reordering |

For startup errors, run `cmus` from a terminal and keep its diagnostic output.
Temporarily move only your hand-written `rc` aside to distinguish a bad option
from an audio/plugin problem; preserve `autosave` and `cache` unless you intend
to rebuild state.

---

## Wide quick reference

| Views | Playback | Navigation | Organize | Modes / command |
| --- | --- | --- | --- | --- |
| `1` tree library | `x` play | `j` / `k` down/up | `a` → library | `r` repeat |
| `2` sorted library | `c` pause | `g` / `G` top/bottom | `y` → playlist | `s` shuffle |
| `3` playlists | `v` stop | `Ctrl-F/B` page | `e` → queue end | `C` continue |
| `4` queue | `b` / `z` next/prev | `/`, `n` / `N` search | `E` → queue front | `m` play scope |
| `5` browser | `B` / `Z` next/prev album | `Tab` next pane | `D` remove | `:` command mode |
| `6` filters | `+` / `-` volume | `Space` mark/toggle | `p` / `P` move | `q` quit |
| `7` settings | `Left` / `Right` seek | `Enter` activate | `u` update | `Ctrl-L` redraw |

---

## References

- [cmus(1) manual](https://man.archlinux.org/man/cmus.1.en)
- [cmus-remote(1) manual](https://man.archlinux.org/man/cmus-remote.1.en)
- [cmus project](https://cmus.github.io/)
- [cmus source repository](https://github.com/cmus/cmus)
