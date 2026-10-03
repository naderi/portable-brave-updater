# Portable Brave Updater

A small Windows program that downloads **Brave** in the channels **Nightly, Beta and Release** (each x86 and x64) and installs it as a **portable version** – no installer, nothing written to `%LOCALAPPDATA%`, one self-contained folder per version.

![Main window](docs/screenshot.png)

## Download

**[Latest release](https://github.com/naderi/portable-brave-updater/releases/latest)** – download `BraveUpdater.exe`, put it into an empty folder where the browsers should go (e.g. `D:\Apps\Brave`) and run it. No installation needed.

**Requirements:** Windows 10 or 11 with .NET Framework 4.5 or later (preinstalled on Windows 10/11).

## How it works

1. Brave publishes every build as a release of the GitHub project [brave/brave-browser](https://github.com/brave/brave-browser/releases). The name starts with the channel, e.g. `Release v1.96.60 (Chromium 154.0.8037.93)`, `Beta v1.97.52 …`, `Nightly v1.98.46 …`.
2. The updater reads the latest ~200 releases through the GitHub API, exactly once per version check. Without signing in, GitHub allows 60 requests per hour.
3. For each channel and architecture the newest release that already contains the **portable ZIP** `brave-v<version>-win32-x64.zip` or `…-win32-ia32.zip` (about 240 MB) is used. Nightly builds are often listed before all files are uploaded; those are skipped.
4. The download is verified by **SHA-256** against the value GitHub provides for the file.
5. The ZIP is extracted with .NET (`brave.exe` + `<version>\…`) and becomes `App\`.
6. `App\` gets read/execute permissions for **ALL APPLICATION PACKAGES** and **ALL RESTRICTED APPLICATION PACKAGES** (see below).

> Brave no longer has a **Dev channel**. The version shown is Brave's own (e.g. `1.96.60`); the underlying Chromium version is part of the release name.
>
> Works on **Windows 10** too: Brave does not need `tar.exe`.

> **Why the permissions?** Brave runs its renderers in an AppContainer sandbox, which may only read folders that are shared with these two groups – as `C:\Program Files` is. Other drives or folders (e.g. `D:\Apps`) usually lack them, and then no page loads and the browser stops responding. The updater sets the permissions on every installation and also checks existing installations at startup. To do it by hand:
> `icacls "<folder>\App" /grant *S-1-15-2-1:(OI)(CI)(RX) *S-1-15-2-2:(OI)(CI)(RX)`

## Using it

| Element | What it does |
|---|---|
| **x86 / x64** per channel | Installs or updates this version. **Green** with a check mark = up to date, **orange** = update available, neutral with a download arrow = not installed. The tooltip shows the folder and the installed version. |
| **Install all: x86 / x64** + **Install all / Update all** | Installs or updates all channels of the selected architectures. Versions that are up to date are skipped. |
| **Create a shortcut on the desktop** | Creates a desktop shortcut after installing (pointing to `BravePortable.exe`). |
| **Add to start menu** | Creates a start menu entry after every installation or update – in the start menu folder **"Brave Portable"** (with *Create a folder for each version*) or as the entry "Brave Portable". Single entries can be removed again via *Extras → Add to start menu*. |
| **Create a folder for each version** | On: every channel gets its own folder (`Brave Stable x64` …). Off: installs directly into the updater's folder; *Install all* is disabled then. |
| **Ignore version check** | Downloads and installs again even if the version is already up to date. |
| **Language (--lang)** | UI language of Brave. Written as `--lang=…` into `Flags=` of every `BravePortable.ini` – immediately for all installed versions and on every further installation. An existing `--lang=…` is replaced; *(no flag – system language)* removes it. |
| **Quit** | Closes the program. During a download it becomes **Cancel**. |
| **ⓘ** (bottom left) | About window with version, developer, GitHub link and program updates. |

**Extras menu**

- *Check versions again* (F5)
- *Open install folder*
- *Create shortcuts on the desktop now* – for all installed versions
- *Add to start menu* – with **Create a folder for each version** a submenu: *All installed versions*, every installed version on its own (check mark = in the start menu, a click adds or removes it) and *Remove all from start menu*. The entries are in the start menu folder **"Brave Portable"**. Without that option it is a single item that adds or removes the entry "Brave Portable".
- *Portable profile for each version* (`.\User Data`) / *One portable profile for all versions* (`..\User Data`) – applied to all installed versions right away
- *Keep downloaded installers* / *Delete downloaded installers* – the downloads are kept in `Update\`; an existing file with the right size and checksum is reused

**Version Info menu:** links to the Brave release notes, the GitHub releases and the about window.

If a Brave instance from the target folder is still running, the updater asks you to close it instead of overwriting files in use. The program appears in German when Windows is set to German, otherwise in English. It follows the light or dark Windows theme and the accent color.

## Folder structure

With **Create a folder for each version**:

```
<updater folder>\
├─ BraveUpdater.exe
├─ BraveUpdater.ini              settings of the updater
├─ User Data\                     only with "one profile for all versions"
├─ Brave Stable x64\
│  ├─ BravePortable.exe        ← start Brave with this
│  ├─ BravePortable.ini        settings of the launcher
│  ├─ User Data\                  portable profile (with "profile for each version")
│  └─ App\
│     ├─ brave.exe …
│     └─ updates\Version.log
└─ …
```

Without that option, `BravePortable.exe`, `BravePortable.ini`, `App\` and `User Data\` are placed directly in the updater's folder.

## The launcher `BravePortable.exe`

Brave has no built-in portable mode. The launcher takes care of it:

- starts `App\brave.exe` with `--user-data-dir=<portable profile>`, so nothing ends up in `%LOCALAPPDATA%`,
- appends the switches from `BravePortable.ini` and its own command line arguments (e.g. a URL),
- carries the icon of its channel.

> The portable Brave does **not update itself**: on Windows, Brave only updates through the separate *BraveUpdate* service, which is never installed here – that is what this updater is for. Features that need an installed Brave (e.g. "make default browser") are not available.

### BravePortable.ini

Next to every `BravePortable.exe`. Lines starting with `;` or `#` are comments.

```ini
UserDataDir=User Data
Flags=--no-default-browser-check --no-first-run --lang=en-US
```

**`UserDataDir`** – folder of the profile (bookmarks, extensions, history, cache …), relative to the launcher's folder or absolute. `User Data` = own profile for this version (default), `..\User Data` = one profile in the updater's folder for all versions.

> The updater rewrites `UserDataDir` on every installation and when you switch it in the *Extras* menu, so better change the profile there. Only the `Flags=` line is kept on updates. A **shared profile** should not be used by two versions at the same time, and a profile of a newer version (e.g. Canary) often cannot be opened by an older one (e.g. Stable) any more.

**`Flags`** – additional command line switches, all **on one line**, separated by spaces; put paths with spaces in quotes. Examples:

| Switch | Effect |
|---|---|
| `--no-default-browser-check` | no question about the default browser |
| `--no-first-run` | skips the welcome page on the first start |
| `--lang=de` | UI language – managed by the updater's **Language** selection |
| `--start-maximized` | starts maximized |
| `--incognito` | starts in incognito / private mode |
| `--disk-cache-dir="R:\Cache"` | puts the cache elsewhere (e.g. a RAM disk); use an absolute path |
| `--proxy-server="socks5://127.0.0.1:1080"` | uses a proxy |
| `--disable-extensions` | disables all extensions |
| `--remote-debugging-port=9222` | remote debugging (e.g. for Puppeteer/Playwright) |

An (unofficial) list of all Chromium switches: <https://peter.sh/experiments/chromium-command-line-switches/>. Arguments passed to `BravePortable.exe` are forwarded too, e.g. `BravePortable.exe https://example.com`.

## Updates of the updater

The updater updates itself from the [releases of this repository](https://github.com/naderi/portable-brave-updater/releases):

- Once a day it looks for a new version in the background (can be switched off in the about window: *Check automatically (once a day)*). If there is one, the ⓘ button gets an orange dot.
- In the about window: *Check for updates* → *Download update* → *Restart & update*.
- A download is only used if its signature (`.sig`) matches the key built into the program; anything else is rejected. The running EXE is renamed to `.old`, the new one takes its name, and the program restarts. `BraveUpdater.ini` and all data next to it stay untouched.

## Disclaimer

This is an independent project. It is not affiliated with, endorsed or sponsored by Brave Software. Brave and its logo are trademarks of their respective owners. The browsers downloaded by this program are subject to their own license terms.

## License

Freeware – free to use, but **not for sale**. See [LICENSE](LICENSE).

© 2026 [Ali Naderi](https://github.com/naderi)
