# Changelog

All notable changes to this project will be documented in this file.

## [2.0.0] - 2026-06-01

### 🐛 Bug Fixes

- **Fixed `\t` escape handling** — Tab escape sequences from logs are now properly processed. Previously `\t` was left as literal characters in the output.
- **Fixed `\\` (double backslash) handling** — Escaped backslashes are now correctly unescaped without corrupting other escape sequences.

### ✨ New Features

- **Side-by-side live preview** — Input and formatted output are displayed in a split-pane layout. Output updates automatically as you type (300ms debounce), no more manual "Convert" button.
- **XML syntax highlighting** — Output panel renders with Catppuccin Frappé color-coded syntax: tags (blue), attributes (yellow), values (green), comments (gray italic), text content (rosewater), CDATA (peach), processing instructions (mauve).
- **Line numbers** — Output panel displays line numbers with hover highlighting.
- **XML error detection** — Malformed XML is detected via `DOMParser`. Error banner shows the error message with line and column number. The offending line is highlighted in the output panel.
- **Valid XML indicator** — A green "✓ valid" badge appears when the XML is well-formed.
- **Auto-extract XML from logs** — Automatically strips surrounding non-XML content (timestamps, log levels, JSON wrappers) from pasted log text.
- **HTML entity decoding** — Automatically decodes `&lt;`, `&gt;`, `&amp;` etc. when the input appears to be HTML-encoded (common in web-based log viewers).
- **Character & line counts** — Both input and output panels show line count and character count.

### 🎨 UI/UX

- **Catppuccin Frappé theme** — Complete redesign using the Catppuccin Frappé color palette. Single dark theme, no light mode.
- **All monospace** — JetBrains Mono font used throughout the entire interface.
- **Editor-style panels** — Panels styled like code editors with subtle borders, hover effects, and indent guides.
- **Improved internal scrolling** — Output lines stay sticky and format neatly like VSCode without growing the main page.
- **Line number highlighting** — Hovering over a code line highlights its corresponding line number.
- **Micro-animations** — Smooth slide-in for error banners, fade-in for badges, press feedback on buttons.
- **Responsive layout** — Panels stack vertically on mobile (< 768px).
- **Custom scrollbars** — Styled to match the Catppuccin theme.
- **Right-aligned toolbar** — Copy and Reset buttons moved to the bottom right for better UX.
- **Changelog Modal** — Added a button to view the changelog directly inside the app.

### 🔄 Changed

- Removed manual "Convert" button — replaced with live formatting.
- Layout changed from vertical stack to horizontal side-by-side.
- Output changed from plain textarea to syntax-highlighted code view.

---

## [1.0.0] - Initial Release

### Features

- Paste XML log string and format/beautify it.
- Handles common escape sequences: `\/`, `\r`, `\n`, `\"`.
- Copy to clipboard button.
- Reset button.
- Built with Vite + Vue 3.
- Dark/light theme via `prefers-color-scheme`.
