# Log XML Formatter

Format and beautify XML logs from Docker, Grafana, or any log source — right in your browser.

> **100% client-side.** Nothing is sent to any server. All processing happens locally in your browser.

## Features

- **Side-by-side live preview** — see formatted XML as you type
- **Syntax highlighting** — Catppuccin Frappé themed color-coded XML
- **Error detection** — shows parse errors with line and column numbers
- **Smart unescaping** — handles `\"`, `\\`, `\n`, `\r`, `\t`, `\/` from log output
- **Auto-extract XML** — strips surrounding log noise (timestamps, log levels, JSON wrappers)
- **HTML entity decoding** — handles `&lt;`, `&gt;`, `&amp;` from web log viewers
- **Copy to clipboard** — one-click copy of formatted output
- **Line numbers** — with error line highlighting
- **Responsive** — works on desktop and mobile

## Tech Stack

- [Vue 3](https://vuejs.org/) — Composition API with `<script setup>`
- [Vite](https://vitejs.dev/) — dev server and build tool
- [xml-formatter](https://www.npmjs.com/package/xml-formatter) — XML formatting
- [JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono) — monospace font
- [Catppuccin Frappé](https://catppuccin.com/) — color theme

## Development

```bash
npm install
npm run dev
```

## Build

```bash
npm run build
npm run preview
```

## Author

Made with ❤️ by [Pandika Pinata](https://id.linkedin.com/in/pandikapinata)

## License

[MIT](LICENSE)
