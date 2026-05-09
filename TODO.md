# Workspace TODO

- [ ] Trading plugin: Add validator action for voucher vs partner-price tag mismatches
	- Context: Voucher item prices can diverge from the effective `partnerprice.unitprice`, and users need a fast consistency check.
	- Goal: Add a new action in the trading plugin that validates price tags by comparing voucher prices against the actual partner-price unit price.
	- Behavior: User runs the action from relevant trading vouchers; system scans lines and detects mismatches between stored voucher price and computed/current `partnerprice.unitprice`.
	- Output: Show a human-friendly, pretty formatted result (grouped list/table) that highlights mismatching rows, expected vs actual values, and deltas.
	- UX expectation: If no mismatches are found, show a concise success message; if mismatches exist, make output readable enough for direct corrective action.
	- Extra note: Keep this as a non-destructive validation/reporting action (no automatic updates), with optional follow-up action(s) considered separately.

- [ ] Android feature: In-app PDF viewer for `window.open` document links in `lino_webview`
	- Context: In `android/lino_android/lino_webview`, PDF links currently trigger the View/Download dialog because Android WebView cannot reliably render PDFs by itself.
	- Goal: Let users read PDFs inside the app (without leaving to external apps) when a `.pdf` is opened from Lino actions.
	- Recommended approach: Download PDF with authenticated session (reuse WebView cookies + user-agent), store temporarily, render with Android native `PdfRenderer` in a dedicated in-app screen.
	- UX expectation: When user chooses **View** for a PDF, open the in-app PDF screen; keep **Download** option unchanged.
	- Fallbacks: If rendering fails, show a clear error and allow external open/download fallback.
	- Extra note: This should support private/authenticated PDFs (not only public URLs).
