# KDE Plasma / Linux

Updated 2026-10-21. The implementation supports two distinct window bindings:

- Native Wayland: KDE `org_kde_kwin_appmenu_manager` associates the D-Bus menu with
  the existing surface. The bundled winit patch exposes a safe cloned proxy;
  winit retains the socket, surface ownership and event dispatch.
- X11/XWayland: KDE window properties plus `com.canonical.AppMenu.Registrar`.

Both use the same `vendor/craft-desktop` transport. `ui-egui::native_menu` derives
commands from the registry and interactive UI entry points; `WordApp::run` remains
the handler. Clicks are revalidated against the current document, modal state and
availability. Lazy submenu state updates use property signals and per-parent layout
signals. The ribbon stays visible, with horizontal overflow and clickable edge
arrows for its tab strip and command groups.

## Build and launch

```sh
cargo build -p wordcraft
WORDCRAFT_WAYLAND=1 cargo run -p wordcraft -- --sample
```

The existing default display policy is unchanged: when XWayland is available it
is preferred for winit file drag-and-drop. The environment override above selects
native Wayland, which also avoids XWayland fractional-scale resampling. File dialogs
remain available on both backends; file drag-and-drop is a separate known limitation.

Add Plasma's **Global Menu** widget to the panel if needed. No GTK module or Qt
conversion is required. Absence of that widget, registrar, session bus or compositor
protocol must not remove local commands. `WORDCRAFT_NO_GLOBAL_MENU=1` or
`KDE_NO_GLOBAL_MENU=1` disables the exporter. On other operating systems the adapter
is not compiled. No compositor configuration is edited by the application.

The root Cargo manifest's winit patch and both vendor directories are required in
source distributions. See `vendor/winit/CRAFT_PATCH.md` for the one-method patch,
upstream checksum and licence. Upstream winit contains its own platform unsafe code;
Craft's new accessor and integration crate contain no unsafe code.

## Appearance and scaling

System mode now reads the standardized portal color-scheme (0/1/2) and accent-color,
updates while idle (two-second polling, bounded calls on a worker), recovers after
portal restart, and stops the worker when its handle is dropped. Explicit Light/Dark
retain WordCraft's palettes. Portal accent adaptation protects text contrast; this
is not a full Qt/Breeze widget skin. Linux window decorations remain compositor-owned.

Native DPI remains separate from Interface Size. For reproducible offscreen checks,
the `ui_shot` script's first JSON line accepts `width`, `height`,
`pixels_per_point` (1, 1.25, 1.5, 1.75, 2) and `integrated_titlebar:false` (Linux SSD).
These are app-test settings, not changes to the desktop scale.

## Reproducible validation

```sh
cargo fmt --all -- --check
cargo check -p wordcraft
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p wordcraft-ui-egui native_menu
cargo test -p wordcraft-ui-egui desktop_
cargo test --manifest-path vendor/craft-desktop/Cargo.toml --features linux,portal
dbus-run-session -- cargo test --manifest-path vendor/craft-desktop/Cargo.toml --features linux,portal appmenu_dbus -- --ignored
```

Use a fresh temporary XDG_CONFIG_HOME and WORDCRAFT_NO_PREFS for synthetic GUI tests;
do not overwrite a real document or clipboard. Check both native Wayland and XWayland:

1. Focus the editor. Confirm textual top-level sections in the Plasma panel, and
   that focus on another app switches the global menu.
2. Toggle Bold through the panel; check the document and checkmark, then Undo/Redo.
   Open a contextual table menu and verify its removal after leaving the table.
3. Open a modal dialog, replace/close the document and reject delayed menu events.
   Close a dirty document and verify the normal save confirmation.
4. Test with no Global Menu widget/exporter disabled and after panel restart.
5. In System mode change the desktop preference and accent; then pin Light/Dark
   and verify they no longer track it. Check popups and dialogs too.
6. Resize at 100/125/150/175/200%; scroll the tabs/groups at narrow widths. Move
   between different-DPI monitors; check text sharpness, fullscreen, SSD buttons,
   drag/resize, system menu and shadows. This requires real monitor/compositor QA.
7. Open/save/cancel a synthetic document through the KDE portal file picker.

Automated protocol/layout tests do not certify visible Plasma rendering, real monitor
transitions, file chooser interaction or sandbox packaging. The workspace's final
`docs/KDE_INTEGRATION_REPORT.md` records which checks were actually run.
