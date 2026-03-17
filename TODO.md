# Workspace TODO

- [ ] Android feature: In-app PDF viewer for `window.open` document links in `lino_webview`
	- Context: In `android/lino_android/lino_webview`, PDF links currently trigger the View/Download dialog because Android WebView cannot reliably render PDFs by itself.
	- Goal: Let users read PDFs inside the app (without leaving to external apps) when a `.pdf` is opened from Lino actions.
	- Recommended approach: Download PDF with authenticated session (reuse WebView cookies + user-agent), store temporarily, render with Android native `PdfRenderer` in a dedicated in-app screen.
	- UX expectation: When user chooses **View** for a PDF, open the in-app PDF screen; keep **Download** option unchanged.
	- Fallbacks: If rendering fails, show a clear error and allow external open/download fallback.
	- Extra note: This should support private/authenticated PDFs (not only public URLs).

- [ ] Android diagnostics: Add WebView service-worker runtime probe in `lino_webview`
	- Context: React service-worker is configured, but we need an on-device proof that it is active/controlling pages in Android WebView.
	- Goal: Log definitive runtime status to Logcat after page load.
	- Probe checks (via JS evaluate):
		- `serviceWorker` API availability (`'serviceWorker' in navigator`)
		- registration count (`navigator.serviceWorker.getRegistrations().length`)
		- control state (`navigator.serviceWorker.controller` present/absent)
		- current page origin/protocol (`location.origin`, `location.protocol`) to confirm secure context
	- Native logs: Prefix with stable tag (e.g. `SW_PROBE`) so logs are easy to filter.
	- Acceptance criteria: On a supported HTTPS origin, logs clearly show whether SW is unavailable, installed-not-controlling, or actively controlling.
