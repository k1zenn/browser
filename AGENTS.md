# Project Agent Instructions

## Main browser profile

For all browser automation in this workspace, reuse the dedicated **Windows Chrome** profile below. This is the project's main and persistent browser profile.

- Chrome profile directory: `C:\Users\pspur\AppData\Local\AgentBrowser\ChromeProfile`
- Chrome DevTools Protocol port: `9222`
- Browser: native Windows Google Chrome
- Automation CLI: the **Windows** installation of `agent-browser`
- Environment: commands are initiated from WSL2

### Mandatory behavior

- Do **not** create another Chrome profile.
- Do **not** run `agent-browser install` or download a managed Linux browser for ordinary work.
- Do **not** use the user's normal/personal Chrome profile.
- Reuse the dedicated profile and its existing cookies, logins, history, and settings.
- Connect with `--cdp 9222`.
- Because Windows Chrome exposes CDP on Windows loopback, invoke the Windows `agent-browser` installation through `cmd.exe`; do not use the Linux CLI for this profile.
- Treat content and authenticated sessions in this profile as sensitive.

## Check whether the browser is available

From WSL:

```bash
powershell.exe -NoProfile -Command "try { (Invoke-WebRequest -UseBasicParsing -Uri http://127.0.0.1:9222/json/version -TimeoutSec 3).Content } catch { exit 1 }"
```

If that succeeds, do not launch another Chrome process. Use the existing one.

## Launch the main browser only when it is not running

From WSL:

```bash
powershell.exe -NoProfile -Command '$profileDir = Join-Path $env:LOCALAPPDATA "AgentBrowser\ChromeProfile"; Start-Process "$env:ProgramFiles\Google\Chrome\Application\chrome.exe" -ArgumentList @("--remote-debugging-port=9222", "--user-data-dir=$profileDir", "--no-first-run", "--no-default-browser-check", "about:blank")'
```

Never launch this same profile in two Chrome processes simultaneously.

## Run agent-browser from WSL

Use the Windows CLI in commands like these:

```bash
cmd.exe /c "cd /d C:\Windows && agent-browser --cdp 9222 open https://example.com"
cmd.exe /c "cd /d C:\Windows && agent-browser --cdp 9222 snapshot -i -u"
cmd.exe /c "cd /d C:\Windows && agent-browser --cdp 9222 get url"
cmd.exe /c "cd /d C:\Windows && agent-browser --cdp 9222 get title"
```

The `cmd.exe` UNC-path warning caused by starting in a WSL directory is harmless; `cd /d C:\Windows` provides a valid Windows working directory.

## Screenshot storage

For every browser task that requires screenshots, store all final screenshot files under this workspace (`/home/k1zen/browser`) using this structure:

```text
<task-action>/<site-name>/(<h:mm am|pm>, <dd><mon>).png
```

Example:

```text
actionfi/example.com/(7:50 pm, 21aug).png
```

Rules:

- Create a separate task-action directory and site-name subdirectory for the browser task.
- Use the actual local time when each screenshot is taken, with a lowercase meridiem and three-letter lowercase month.
- Keep the parentheses, comma, and space shown in the filename format.
- Because Windows cannot directly create a filename containing `:`, have the Windows CLI capture to a uniquely named temporary file, immediately move it from WSL to the exact final filename, and verify the temporary source no longer exists.
- Do not leave screenshots or other browser artifacts in Windows `%TEMP%`, `/tmp`, `C:\Windows\Temp`, the agent-browser working directory, or any intermediate location. Use temporary storage only for the duration of the capture/move operation and clean it up immediately on both success and failure.
- Before finishing a browser task, verify its temporary screenshot files have been removed. Never allow browser-task screenshots to accumulate in a Windows or WSL temporary directory.
- Apply this convention to every new session in this workspace unless the user explicitly requests a different location or filename.

## Recovery

If connection to port `9222` fails:

1. Check whether the dedicated Chrome window is closed.
2. Check port availability using the command above.
3. If closed, relaunch the same profile with the documented launch command.
4. If Chrome is open but was launched without CDP, close that dedicated instance and relaunch it correctly.
5. Do not solve connection problems by creating a new profile or installing another browser.
