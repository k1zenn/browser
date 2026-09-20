# Project Agent Instructions

## Environment

This workspace now runs on **Windows native**, not WSL. Commands are plain Windows
commands: no `powershell.exe` / `cmd.exe /c` wrappers, no UNC-path warnings.

The workspace root is this directory. Everything below is relative to it.

## Main browser profile

For all browser automation, reuse the dedicated **Windows Chrome** profile below. This is
the project's main and persistent browser profile.

- Chrome profile directory: `C:\Users\pspur\AppData\Local\AgentBrowser\ChromeProfile`
- Chrome DevTools Protocol port: `9222`
- Browser: native Windows Google Chrome
- Automation CLI: the **Windows** installation of `agent-browser`
- Optional secondary agent: the vendored `jev-browser/` (Jev-driven, see below)

### Mandatory behavior

- Do **not** create another Chrome profile.
- Do **not** run `agent-browser install` or download a managed browser for ordinary work.
- Do **not** use the user's normal/personal Chrome profile.
- Reuse the dedicated profile and its existing cookies, logins, history, and settings.
- Connect with `--cdp 9222`.
- Treat content and authenticated sessions in this profile as sensitive.

## Check whether the browser is available

```powershell
try { (Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:9222/json/version -TimeoutSec 3).Content } catch { exit 1 }
```

Or with `curl.exe`, which ships with Windows:

```cmd
curl.exe -s --max-time 3 http://127.0.0.1:9222/json/version
```

If either succeeds, do not launch another Chrome process. Use the existing one.

## Launch the main browser only when it is not running

```powershell
$profileDir = Join-Path $env:LOCALAPPDATA "AgentBrowser\ChromeProfile"
Start-Process "$env:ProgramFiles\Google\Chrome\Application\chrome.exe" -ArgumentList @("--remote-debugging-port=9222", "--user-data-dir=$profileDir", "--no-first-run", "--no-default-browser-check", "about:blank")
```

Never launch this same profile in two Chrome processes simultaneously.

## Run agent-browser

Run the Windows CLI directly:

```cmd
agent-browser --cdp 9222 open https://example.com
agent-browser --cdp 9222 snapshot -i -u
agent-browser --cdp 9222 get url
agent-browser --cdp 9222 get title
```

## Jev browser (optional)

`jev-browser/` is a vendored, CDP-patched copy of
[jkudish/jev-browser](https://github.com/jkudish/jev-browser): a Jev-driven agent that picks
one action per step from the page's interactive elements. See `jev-browser/PATCH.md`.

Use it when a task is better expressed as "reach this goal on this site" than as explicit
steps. It attaches to the same port-9222 profile, opens its own tab, and closes only that tab.

```powershell
cd jev-browser
npm install                       # builds dist/; set $env:JEV_BROWSER_SKIP_BROWSER_DOWNLOAD = "1" to skip the Chromium download
$env:TYPESAFE_API_KEY = "ts_..."  # from console.typesafe.ai/settings/keys
npx jev-browser run "Find the price of the Pro plan" https://example.com/pricing --cdp http://127.0.0.1:9222
```

- It never closes the attached Chrome; it only closes the tab it opened.
- `--record` is ignored over CDP.
- Typing needs a second, text-generating model (Jev returns decisions only). Without one it
  falls back to a keyword heuristic. Configure with `JEV_BROWSER_TYPE_*`.
- `TYPESAFE_API_KEY` is not stored in this repo. Never read, print, log, or commit it.

## Screenshot storage

Store all final screenshot files under this workspace using this structure:

```text
<task-action>/<site-name>/(<h.mm am|pm>, <dd><mon>).png
```

Example:

```text
actionfi/example.com/(7.50 pm, 21aug).png
```

Rules:

- Create a separate task-action directory and site-name subdirectory for the browser task.
- Use the actual local time when each screenshot is taken, with a lowercase meridiem and
  three-letter lowercase month.
- Keep the parentheses, comma, and space shown in the filename format.
- Use a **dot** between hours and minutes. Windows (NTFS) forbids `:` in filenames, so the
  original `7:50 pm` form cannot be created on this filesystem. If you specifically need the
  literal-colon form, write the file into the WSL filesystem instead and do the final rename
  from WSL.
- Do not leave screenshots or other browser artifacts in `%TEMP%`, `C:\Windows\Temp`, the
  agent-browser working directory, or any other intermediate location. Use temporary storage
  only for the duration of the capture/move operation and clean it up immediately on both
  success and failure.
- Before finishing a browser task, verify its temporary screenshot files have been removed.
  Never allow browser-task screenshots to accumulate in a temporary directory.
- Apply this convention to every new session unless the user explicitly requests a different
  location or filename.

## Recovery

If connection to port `9222` fails:

1. Check whether the dedicated Chrome window is closed.
2. Check port availability using the command above.
3. If closed, relaunch the same profile with the documented launch command.
4. If Chrome is open but was launched without CDP, close that dedicated instance and relaunch
   it correctly.
5. Do not solve connection problems by creating a new profile or installing another browser.

## If you are running from WSL instead

The above is written for Windows native. From WSL, wrap Windows commands explicitly:

```bash
powershell.exe -NoProfile -Command "try { (Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:9222/json/version -TimeoutSec 3).Content } catch { exit 1 }"
cmd.exe /c "cd /d C:\Windows && agent-browser --cdp 9222 open https://example.com"
```

The `cmd.exe` UNC-path warning caused by starting in a WSL directory is harmless;
`cd /d C:\Windows` provides a valid Windows working directory.
