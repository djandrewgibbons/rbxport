# rbxport for macOS: SwiftUI port plan

The goal is a macOS-only rbxport with a native SwiftUI interface and the same
feature set as the Tauri app, with no change to library or export behavior.

## Decisions

| Topic | Decision | Reason |
| --- | --- | --- |
| Platform | macOS only (14 Sonoma or later) | SwiftUI is Apple-only. macOS 14 gives `@Observable`, `Table` improvements and `Inspector`. |
| Rust crates | Keep all 18 `rbl-*` crates unchanged | About 75k lines of reverse-engineered format and protocol code (SQLCipher `master.db`, `export.pdb`, OneLibrary, ANLZ, PRO DJ LINK, NFS, dbserver). Rewriting it in Swift adds risk and no user value. |
| Tauri shell (`src-tauri/`) | Leave in place, do not edit. Port its logic into a new Tauri-free crate | Keeps upstream merges clean. `src-tauri` can be deleted from this branch once the Swift app reaches parity. |
| React frontend (`src/`) | Replace with SwiftUI. Leave in place until parity | Same reason as above. It also serves as the behavior reference. |
| Rust to Swift bridge | [UniFFI](https://mozilla.github.io/uniffi-rs/) | Generates Swift types, errors, async functions and callback interfaces from Rust. No hand-written C headers. |
| Packaging | Rust built as a static library inside an XCFramework, consumed by a local Swift package | Standard Xcode integration. One `cargo` build script per architecture, merged with `lipo`. |
| Distribution | Developer ID signed and notarized DMG. Not the Mac App Store | rbxport is GPL-2.0-or-later and links Rubber Band (GPL). App Store terms conflict with the GPL. Raw USB formatting and the LINK servers also need entitlements the App Store sandbox does not allow. |
| Updates | [Sparkle 2](https://sparkle-project.org) | Replaces `tauri-plugin-updater`. |
| Crash reporting | `sentry-cocoa` (optional, off by default) | Replaces the Tauri Sentry hook. Rust panics already go through `sentry` in Rust. |

## Target architecture

```
SwiftUI views
   │  @Observable models (LibraryModel, DeckModel, LinkModel, ...)
   ▼
RbxKit (Swift package)           ← generated UniFFI bindings + thin Swift helpers
   │  UniFFI (C ABI)
   ▼
crates/rbx-ffi                   ← #[uniffi::export] API, event callback interface
   ▼
crates/rbx-app                   ← Tauri-free port of src-tauri: AppState, edit path,
   │                               link, backups, relocate, cues, grid, analysis jobs
   ▼
crates/rbl-*                     ← unchanged
```

### New directories

| Path | Contents |
| --- | --- |
| `crates/rbx-app/` | App core ported from `src-tauri/src`. No `tauri` dependency. Exposes plain Rust structs and an `EventSink` trait. |
| `crates/rbx-ffi/` | UniFFI surface over `rbx-app`. Owns DTO conversion and the event callback. Opts out of the workspace `unsafe_code = "deny"` lint because UniFFI scaffolding needs it. |
| `macos/RbxKit/` | Swift package: generated bindings, `RbxCore.xcframework`, async wrappers, `Backend` protocol and a mock backend. |
| `macos/rbxport/` | Xcode app target: SwiftUI views, models, menus, AppleScript, Sparkle. |
| `macos/scripts/` | `build-rust.sh` (cargo for arm64 and x86_64, lipo, xcframework, uniffi-bindgen). `locales.mjs` (JSON locales to `.xcstrings`). |

The root `Cargo.toml` already includes `crates/*`, so the new crates join the
workspace without editing it.

### Event model

The Tauri shell emits 19 events. `rbx-ffi` defines one callback interface:

```rust
#[uniffi::export(callback_interface)]
pub trait EventListener: Send + Sync {
    fn on_event(&self, event: AppEvent);
}
```

`AppEvent` is a UniFFI enum with one variant per event (`LibraryChanged { generation }`,
`DeckTick { .. }`, `ExportProgress { .. }`, ...). In Swift, `RbxKit` fans it out
into `AsyncStream`s that the models consume on the main actor.

High-rate events (`deck:tick`, `deck:meters`) do not cross per frame. Swift
polls a shared snapshot from a `CADisplayLink` (`NSView.displayLink`) instead,
matching how the React player reads deck state.

| Tauri event | Swift consumer |
| --- | --- |
| `library:ready`, `library:changed`, `library:problem` | `LibraryModel` |
| `analysis:changed`, `cues:changed`, `grid:changed` | `TrackDetailModel`, `PlayerModel` |
| `deck:loaded`, `deck:reset`, `deck:error`, `deck:tick`, `deck:meters` | `DeckModel` (tick and meters polled) |
| `devices:changed`, `export:progress`, `export:done`, `sync:progress`, `import:progress` | `DevicesModel`, `SyncModel` |
| `link:status`, `link:peers` | `LinkModel` |
| `update:progress` | Removed. Sparkle owns updates. |

## Command inventory

The Tauri app registers about 186 commands. Each one maps to an `rbx-app`
function and an `rbx-ffi` export. Grouped by area:

| Area | Commands | Source module |
| --- | --- | --- |
| Library open/create/switch | `library_summary`, `disable_read_only`, `reload_library`, `library_problem`, `create_library`, `use_default_library`, `database_drives`, `switch_library`, `rekordbox_browse_settings` | `commands.rs`, `new_library.rs`, `browse_settings.rs` |
| Browsing | `playlist_tree`, `open_view`, `fetch_rows`, `view_ids_in_range`, `filter_values`, `explorer_roots`, `explorer_children` | `commands.rs`, `explorer.rs` |
| Playlists and edits | `create_playlist`, `create_folder`, `create_smart_playlist`, `smart_rule`, `set_smart_rule`, `rename_playlist`, `move_playlist`, `delete_playlist`, `reorder_playlist`, `add_tracks_to_playlist`, `remove_tracks_from_playlist`, `playlist_duplicates`, `undo_edit`, `redo_edit` | `commands.rs` |
| Track metadata | `track_details`, `selection_details`, `track_lookups`, `set_track_field`, `set_track_rating`, `set_track_comment`, `set_track_color`, `set_my_tags`, artwork (4), `reset_play_count`, `record_play`, `remove_from_history`, `remove_from_collection` | `details.rs`, `commands.rs` |
| Tag List | `reload_tags`, `add_to_tag_list`, `remove_from_tag_list`, `clear_tag_list` | `commands.rs` |
| Cues and grid | 8 cue commands, `grid_state`, `grid_edit`, `grid_undo`, `grid_redo`, `grid_lock`, `track_cues`, `track_beats`, `track_phrases`, `track_vocals` | `cues.rs`, `grid.rs` |
| Analysis | `analyse_track`, `edit_phrase`, `unanalysed_tracks`, `track_waveform`, `track_pcm_waveform` | `analysis.rs`, `commands.rs` |
| Decks and mixer | 20 `deck_*` commands, `set_crossfade`, `set_channel_*` (3), `set_eq_curve`, `set_master_level`, `master_limiter`, `set_master_limiter`, `set_metronome`, `audio_devices`, `set_audio_device`, `set_audio_config`, `preview_*` (3), `export_loop_wav` | `commands.rs`, `player.rs`, `preview.rs` |
| USB export and sync | `list_devices`, `export_playlist`, `export_progress`, `cancel_export`, `sync_devices`, `estimate_sync`, `validate_export_files`, `eject_device`, `format_device`, `export_tracks_to_device`, `device_sync_state`, `import_usb`, `device_settings` (5), `device_libraries`, `device_playlist_edit` | `commands.rs`, `device_*.rs`, `usb_import.rs` |
| PRO DJ LINK | `link_status`, `link_peers`, `start_link_export`, `stop_link_export`, `link_load_track`, `link_set_master`, `link_nudge_master`, `link_take_master_tempo` | `link.rs` |
| Import/export files | `import_files`, `import_folder_playlist`, `list_import_files`, `import_files_slice`, `import_xml`, `export_xml`, `import_itunes*` (3), `export_playlist_file`, `find_duplicates` | `commands.rs` |
| Missing files | `missing_tracks`, `remove_missing_tracks`, `relocate_track`, `auto_relocate`, `relocation_targets`, `relocate_by_location` | `relocate.rs` |
| Backups | `list_backups`, `backup_directory`, `backup_sizes`, `backup_progress`, `start_backup`, `cancel_backup`, `back_up_library`, `delete_backup`, `set_backup_directory`, `open_backup_directory` | `backups*.rs` |
| App and OS | `app_version`, `app_diagnostics`, `open_log`, `reveal_track`, `open_url`, menus (2), windows (4), file drag/drop (2), updater (4), report (3), scripting (3), `startup_*` (2), `remember_screen_assets`, `test_eval_result` | Replaced by native Swift. Not ported to Rust. |

## Screen parity checklist

Each row is done when the SwiftUI screen matches the React screen's behavior,
keyboard shortcuts and localized strings.

| # | Screen | React source | SwiftUI approach |
| --- | --- | --- | --- |
| 1 | Main window layout, layout menu | `app/App.tsx`, `topbar/*` | `NavigationSplitView` with sidebar, content and `.inspector`. Toolbar for top bar. |
| 2 | Source rail and playlist tree | `tree/SourceRail.tsx`, `tree/TreeView.tsx` | `List` with `OutlineGroup`, drag to reorder, rename in place, context menus. |
| 3 | Smart playlist editor | `tree/SmartPlaylistEditor.tsx` | Sheet with a rule builder form. |
| 4 | Track table | `browser/TrackTable.tsx`, `ColumnMenu.tsx`, `dragGhost.ts` | `NSTableView` in `NSViewRepresentable`, paged from `fetch_rows`. SwiftUI `Table` does not hold 100k+ rows at the current frame budget. Column chooser, sort, multi-select, drag out as file URLs and track IDs. |
| 5 | Track filter and search | `browser/TrackFilter.tsx`, `components/SearchField.tsx` | Toolbar `.searchable` plus filter popover. Filtering stays in `rbl-index`. |
| 6 | Waveform preview column and right rail | `browser/WaveformPreview.tsx`, `RightRail.tsx`, `canvas/waveform.ts` | Core Graphics drawing in a reusable `WaveformView`, cached as `CGImage`. |
| 7 | Sub-browser | `subbrowser/SubBrowser.tsx` | Second paged table in a split. |
| 8 | Info panel and field editing | `info/InfoPanel.tsx`, `info/fields.ts` | `.inspector` with `Form`, artwork drop target, My Tag editor. |
| 9 | Player: dual deck, simple player | `player/Player.tsx`, `DualDeck.tsx`, `SimplePlayer.tsx` and helpers | `DeckView` per deck. Detail waveform, hot cues, memory cues, loops, jump menu, key shift, tempo slider and field, time readouts. Metal or Core Animation layers for the scrolling waveform. |
| 10 | Mixer strip, volume knob, VU | `player/MixerStrip.tsx`, `topbar/VolumeKnob.tsx`, `components/VuMeterFill.tsx` | Custom knobs and meters driven by the polled deck snapshot. |
| 11 | Grid and phrase editor | `player/useGridEditor.ts`, `VocalStrip.tsx` | Overlay on the detail waveform. |
| 12 | Analysis dialog | `analysis/AnalysisDialog.tsx` | Sheet with progress list. |
| 13 | Missing file manager, new library dialog | `library/*` | Window and sheet. |
| 14 | Devices panel (General, Color, Column, List pair, My Settings) | `devices/*` | Settings-style `TabView` per device. |
| 15 | Sync manager, format device, missing files | `sync/*` | Separate `Window` scene, matching the current separate sync window. |
| 16 | Status bar, LINK deck strip | `statusbar/*` | Bottom bar in the main window. |
| 17 | Preferences (12 panes) | `settings/*` | `Settings` scene with one tab per pane. Backup size chart with Swift Charts. |
| 18 | Update manager | `update/*` | Sparkle standard UI. |
| 19 | Report a bug | `report/ReportBug.tsx` | Separate window, attachments built by `rbx-app`. |
| 20 | Context menus, confirm dialogs, duplicates dialog | `components/*` | `.contextMenu`, `.confirmationDialog`, sheet. |

### Native features replacing Tauri plugins

| Feature | Tauri today | macOS port |
| --- | --- | --- |
| Menus | `menu.rs` | SwiftUI `Commands`, including the History menu. |
| Keyboard shortcuts | `settings/KeyboardPane.tsx` and frontend handlers | `.keyboardShortcut` plus a custom key map stored in `UserDefaults`, honoring user remapping. |
| AppleScript | `rbxport.sdef`, `scripting/cocoa.rs`, `scripting/model.rs` | Keep `rbxport.sdef`. Implement `NSScriptCommand` subclasses in Swift calling the same `rbx-app` model. |
| File drag out | `file_drag.rs` (objc2 pasteboard) | `Transferable` / `NSPasteboardItem` with file URLs. |
| File drop in | `file_drop.rs` | `.dropDestination(for: URL.self)`. |
| Dialogs | `tauri-plugin-dialog` | `NSOpenPanel`, `NSSavePanel`, `.fileImporter`. |
| Open URL, reveal in Finder | `tauri-plugin-opener` | `NSWorkspace`. |
| Window state | `tauri-plugin-window-state`, `windowfit.rs` | SwiftUI scene restoration and `defaultSize`. |
| Elevated updates, format device | `elevated_update.rs`, `format_device` | Sparkle for updates. Format through `diskutil` from `rbx-app` (unchanged logic). |
| Localization | `public/locales/*.json`, `src/i18n` | `Localizable.xcstrings` generated from the same JSON by `macos/scripts/locales.mjs`. |

## Phases

| Phase | Scope | Done when |
| --- | --- | --- |
| 0. Toolchain | `rbx-ffi` hello-world export, `build-rust.sh`, XCFramework, empty SwiftUI app calling Rust | App launches on Apple Silicon and Intel and shows `app_version` from Rust. |
| 1. App core | Create `rbx-app`: port `state.rs`, `dto.rs`, `error.rs`, the edit path from `commands.rs`, `durable.rs`, `file_journal.rs`. Replace `AppHandle` emits with `EventSink`. Port the shell's Rust tests. | `cargo test -p rbx-app` passes with the same fixtures as `src-tauri/tests`. |
| 2. Read-only browser | Screens 1, 2, 4, 5, 7, 8 (read only), 16 (library part) | Opens a real library read-only, scrolls 100k tracks at 60 fps on Apple Silicon, search and sort match the Tauri app. |
| 3. Editing | Playlist and metadata edits, Tag List, undo/redo, smart playlists, artwork, screens 3 and 20 | Each edit produces the same `master.db` diff as the Tauri app (checked with `rbl-difftool`). |
| 4. Playback | Decks, mixer, preview, cues, grid, analysis: screens 6, 9, 10, 11, 12 | Deck latency and scrub match the Tauri app. No dropped audio while scrolling the table. |
| 5. USB and devices | Export, sync, format, eject, device settings, USB import: screens 14, 15 | Export of the shared test library is byte-identical to the Tauri app's export. |
| 6. PRO DJ LINK | Link servers, peers, master tempo, LINK deck strip | Real CDJ/XDJ browses and loads tracks; `rbl-fakecdj` loopback tests pass. |
| 7. Library tools | Missing files, relocate, import (files, XML, iTunes), export XML, backups and restore, duplicates: screens 13, 17 | Feature-by-feature check against the Tauri app. |
| 8. Native integration | Menus, shortcuts, AppleScript, drag and drop, Sparkle, Sentry, report window, localization: screens 18, 19 | All locales load. AppleScript suite passes the existing scripting tests. |
| 9. Release | Signing, notarization, DMG, macOS CI workflow, remove `src/` and `src-tauri/` from this branch | A notarized DMG installs and updates through Sparkle. |

## Testing

| Layer | Tooling |
| --- | --- |
| Rust core | `cargo test` on `rbx-app` and `rbx-ffi`, reusing `rbl-db` fixtures. Runs on Linux and macOS. |
| Swift models | XCTest against the mock `Backend` (port of `src/ipc/backend-mock.ts`). |
| UI | XCUITest for the main flows: open library, edit, export, play. |
| Parity | Same-input runs of the Tauri app and the Swift app compared with `rbl-difftool` (database writes) and byte comparison (USB exports). |
| Performance | Instruments signposts for first rows and scroll frame cost, with gates mirroring `perf-budgets.json`. |

## Keeping up with upstream

- Keep `main` in sync with the upstream repository.
- Merge `main` into the SwiftUI branch regularly. Changes in `crates/rbl-*` merge cleanly because this branch does not edit them.
- Changes upstream makes in `src-tauri/src` must be ported by hand into `rbx-app`. Track the last ported upstream commit in `crates/rbx-app/UPSTREAM` so each merge shows the `src-tauri` diff to port.
- Changes in `src/` are behavior references for the matching SwiftUI screen.

## Requirements

- A Mac with Xcode 16 or later for the Swift side. The Rust crates build and test on Linux.
- An Apple Developer ID for signing and notarization (phase 9).
- GitHub Actions is disabled on the fork. Phase 9 adds one macOS workflow when Actions is turned back on.
