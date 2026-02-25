# Workspace TODO

- [ ] Android feature: In-app PDF viewer for `window.open` document links in `lino_webview`
	- Context: In `android/lino_android/lino_webview`, PDF links currently trigger the View/Download dialog because Android WebView cannot reliably render PDFs by itself.
	- Goal: Let users read PDFs inside the app (without leaving to external apps) when a `.pdf` is opened from Lino actions.
	- Recommended approach: Download PDF with authenticated session (reuse WebView cookies + user-agent), store temporarily, render with Android native `PdfRenderer` in a dedicated in-app screen.
	- UX expectation: When user chooses **View** for a PDF, open the in-app PDF screen; keep **Download** option unchanged.
	- Fallbacks: If rendering fails, show a clear error and allow external open/download fallback.
	- Extra note: This should support private/authenticated PDFs (not only public URLs).
