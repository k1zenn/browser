# Local patch: CDP attach

This directory is a vendored copy of [`jkudish/jev-browser`](https://github.com/jkudish/jev-browser),
patched so a run can **attach to an already-running Chrome over CDP** instead of launching
Playwright's own Chromium. That lets it drive the dedicated `AgentBrowser\ChromeProfile`
on port `9222` and reuse that profile's cookies, logins, and extensions.

- Upstream: `https://github.com/jkudish/jev-browser`
- Base version: `0.4.1`
- Vendored: source only (`src/`, `test/`, `scripts/`, manifests, docs). Upstream `assets/`,
  `.github/`, and `.agents/` are omitted.

## What changed

| File | Change |
| --- | --- |
| `src/navigate.ts` | New `NavigateOptions.cdpUrl`; attach via `chromium.connectOverCDP` when set; reuse the browser's existing context; close only the run's tab when attached; skip `recordDir` over CDP. |
| `src/cli.ts` | New `--cdp <url>` flag and help text. |
| `src/index.ts` | New optional `cdp_url` MCP tool argument. |
| `README.md` | Documented `JEV_BROWSER_CDP_URL` and the CDP differences. |

Selection order: `--cdp` / `cdp_url` argument → `$JEV_BROWSER_CDP_URL` → normal launch.

## Diff

```diff
--- src/navigate.ts
+++ src/navigate.ts
@@ -35,6 +35,10 @@
   maxChars?: number;
   screenshot?: "final" | "none";
   recordDir?: string;
+  /** Attach to an already-running Chrome over CDP (e.g. http://127.0.0.1:9222)
+   *  instead of launching a browser, so the run reuses that profile's cookies
+   *  and logins. Defaults to $JEV_BROWSER_CDP_URL. */
+  cdpUrl?: string;
 }
 
@@ -340,8 +344,21 @@
   let browser: Browser | null = null;
+  let attached = false;
+  let runPage: Page | null = null;
   let status = "error";
 
+  // When attached over CDP, `browser` is the user's already-running Chrome.
+  // Closing it would kill their session, so only close the tab this run opened.
+  const shutdown = async () => {
+    if (!browser) return;
+    if (attached) {
+      await runPage?.close().catch(() => {});
+      return;
+    }
+    await browser.close().catch(() => {});
+  };
+
@@ -367,14 +384,32 @@
-    browser = await chromium.launch({ headless: process.env.JEV_BROWSER_HEADED !== "1" });
-    const context = await browser.newContext({
-      viewport: { width: 1024, height: 640 },
-      ...(options.recordDir ? { recordVideo: { dir: options.recordDir } } : {}),
-    });
+    const cdpUrl = options.cdpUrl ?? process.env.JEV_BROWSER_CDP_URL;
+    attached = Boolean(cdpUrl);
+    if (attached && options.recordDir) {
+      extractionProblems.push("recordDir ignored: video recording is not available over CDP");
+    }
+    browser = attached
+      ? await chromium.connectOverCDP(cdpUrl as string)
+      : await chromium.launch({ headless: process.env.JEV_BROWSER_HEADED !== "1" });
+    const context = attached
+      ? browser.contexts()[0]
+      : await browser.newContext({
+          viewport: { width: 1024, height: 640 },
+          ...(options.recordDir ? { recordVideo: { dir: options.recordDir } } : {}),
+        });
+    if (!context) {
+      throw new Error(`No browser context at ${cdpUrl}; is Chrome running with --remote-debugging-port?`);
+    }
     context.setDefaultTimeout(8_000);
     let page = await context.newPage();
+    runPage = page;
@@ -563,7 +598,7 @@
-    await browser.close().catch(() => {});
+    await shutdown();
@@ -600,7 +635,7 @@
-    await browser?.close().catch(() => {});
+    await shutdown();
```

`src/cli.ts` adds `--cdp <url>` to `parseArgs` and `HELP`; `src/index.ts` adds the optional
`cdp_url` string to the tool schema and forwards it as `cdpUrl`.

## Behaviour over CDP

- Opens its own tab; closes **only that tab** when the run ends. Never closes the attached browser.
- `--record` is ignored (video capture is unavailable over CDP) and reported under `extraction_problems`.
- The viewport is the window's own; the 1024x640 default does not apply.
- The browser's existing context is reused, because Playwright cannot create a new one over CDP.

## Re-applying on a new upstream version

```bash
git clone --depth 1 https://github.com/jkudish/jev-browser.git /tmp/jev-upstream
diff -u /tmp/jev-upstream/src/navigate.ts src/navigate.ts
```

Then re-apply the four hunks above and re-run `npm run build && npm test`.
