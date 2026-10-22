# Porting Foliate to Windows 10 & 11 — Analysis

No code has been changed; this is an analysis only.

**Summary:** a Windows port can't start until the app's web engine is replaced, because WebKitGTK, which renders every book, doesn't run on Windows. Most other Linux-specific parts are small fixes.

## What Foliate is built on

- **Language and toolkit:** about 5,700 lines of JavaScript run by GJS (GNOME's JavaScript runtime), using GTK 4.12+ and libadwaita 1.8+, with 21 GTK `.ui` layout files.
- **Book rendering:** WebKitGTK 6.0 renders all book content. The actual e-book engine is `foliate-js`, plain browser JavaScript kept in a git submodule (`src/foliate-js`).
- **Build and packaging:** Meson builds it, and it ships as Flatpak, Snap and Debian packages, all Linux-only.

## The blocker: WebKitGTK

The WebKit project has a Windows version, but it has no GTK widget and no way to be called from GJS. As far as known, WebKitGTK has no maintained Windows build, MSYS2 included (verify before deciding). Foliate uses it in five places:

- `src/webview.js`: custom `foliate:` / `foliate-opds:` / `foliate-selection-tool:` URL schemes, plus `evaluate_javascript` and script message handlers to talk to the page.
- `src/book-viewer.js`, `src/library.js` (the OPDS catalog browser and downloads), `src/selection-tools.js` (translate, Wikipedia and Wiktionary lookups) and `src/image-viewer.js`.
- Inside the pages, `src/reader/reader.js`, `src/opds/main.js` and `src/selection-tools/translate.html` post messages back through `webkit.messageHandlers`.

### Three ways around it

| Option | What it involves | Effort |
|---|---|---|
| **A. Keep GJS/GTK4, swap in Microsoft's WebView2** | Write a small C library that hosts WebView2 inside a GTK4 window and exposes it to GJS, then rewrite `webview.js` on top of it. GJS, GTK4 and libadwaita themselves are available through MSYS2 (a Windows packaging environment). | High and risky. WebView2 sits in its own native child window, so GTK elements meant to overlap the book page may be hidden behind it, and focus/keyboard handling has to be done by hand. |
| **B. Rewrite the app shell on Tauri or Electron** | `foliate-js` runs unchanged in Chromium/WebView2. The library, sidebar, annotations, preferences, text-to-speech and OPDS screens are rewritten as web UI. Readest is an existing reader built on Tauri + foliate-js and is a useful reference. | High but predictable. It's effectively a new app that shares the engine; the GTK UI can't be reused. |
| **C. Run the Linux build under WSLg** | Windows 11 and recent Windows 10 builds can run Linux GUI apps directly, so the current Flatpak would just work. | Almost none, but it isn't a native app. Fine as a stopgap or for testing. |

**Recommendation:** for a native, maintainable Windows app, go with B. Choose A only if staying one codebase with Linux matters more than effort and risk.

## Smaller changes needed regardless of the option

Under option B these become design requirements for the rewrite rather than code edits.

### 1. Startup and paths (`src/main.js`, `src/meson.build`)

- The `#!@GJS@ -m` first line doesn't work on Windows, so the app needs a launcher (a small `.exe`, or a shortcut running `gjs.exe -m`).
- `@datadir@` is fixed at build time, and the translation folder falls back to `/usr/share`. The install has to find its files relative to the program instead, because users can install it anywhere.
- GSettings (stored preferences) needs its compiled settings schema shipped next to the app.

### 2. Text-to-speech (`src/speech.js`)

- It starts the `speech-dispatcher` program and connects over a Unix socket (`Gio.UnixSocketAddress`); neither exists on Windows.
- It needs a new backend using Windows' own speech engines (SAPI or `Windows.Media.SpeechSynthesis`), or the browser's `speechSynthesis` once on WebView2.
- `src/tts.js` should keep its current interface (the SSML handling and marks/events) so the rest of the app doesn't change.

### 3. Tracker (`src/library.js:83`)

- It uses Tracker (GNOME's file indexer) over D-Bus to find moved books. This is Linux-only and must be disabled or guarded on Windows.

### 4. Other Linux-only assumptions

- `src/format.js:15` reads the GNOME clock-format setting. It already fails safely, but should use the Windows regional settings instead.
- `src/app.js:27-28`: the "About / debug info" text prints Linux desktop environment variables (`XDG_*`). Cosmetic only.
- `src/data.js:97` and `src/library.js:171` save book locations with `~` standing for the home folder. This needs testing with drive letters, backslashes, case-insensitive paths and `file:///C:/...` addresses.
- `app.inhibit(IDLE)` keeps the screen awake while reading. Check that it works on Windows; if not, use `SetThreadExecutionState`.
- Check that libadwaita follows the Windows light/dark mode setting; if not, read it from the registry.
- The default font names (Serif/Sans) need checking under Windows fonts.

### 5. Desktop integration

- The Linux desktop entry, metainfo file, icon cache and desktop database steps are Linux-only and must be skipped on Windows.
- They need Windows equivalents:
  - File associations for `.epub`, `.mobi`, `.azw3`, `.fb2`, `.cbz` and `.pdf`.
  - An `opds:` link handler, which `src/app.js:418` already supports.
  - A Start-menu shortcut and an `.ico` icon.
- Opening a book from Explorer while Foliate is running should reuse the open window. GTK uses D-Bus for that, which doesn't exist on Windows, so it needs verifying (option A) or a single-instance plugin (Tauri).

### 6. Packaging and CI

- Option A means bundling the GTK/GJS/libadwaita libraries and their supporting files (icons, image loaders, translations) from MSYS2.
- Both options need an installer (MSIX, Inno Setup or WiX), code signing, and a check that the WebView2 runtime is installed.
- A Windows build job in GitHub Actions; the current workflows are `deb.yml`, `snap.yml` and `eslint.yml`.

## Windows 10 vs 11

- GTK4 and WebView2 both support Windows 10 and 11.
- WebView2 comes with Windows 11. Most Windows 10 machines have it too, but the installer should include Microsoft's small WebView2 setup program just in case.
- Windows 10 reached end of support in October 2025, so Windows 11 should be the main target. Test Windows 10 22H2 for anyone still on extended updates.
- WSLg (option C) needs Windows 11 or Windows 10 build 19044 or later.

## Suggested order of work

1. Decide between A and B by building a quick prototype that loads `foliate-js`'s `reader.html` in WebView2, connects the message passing, and opens a book.
2. Fix the startup and paths (item 1).
3. Port the web view layer: the custom URL schemes, page messaging, downloads, link handling, printing and crash handling.
4. Replace text-to-speech (item 2) and switch off Tracker (item 3).
5. Fix the remaining Linux assumptions (item 4).
6. Add desktop integration (item 5).
7. Set up the installer, signing and CI (item 6).
