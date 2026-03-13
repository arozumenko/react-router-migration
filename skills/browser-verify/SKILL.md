---
name: browser-verify
description: "Use when asked to verify in browser, check the page, test in Chrome, take a screenshot, verify the UI, check if it renders, open in browser, browser test, visual verification, check the app, verify the layout, test the form, check for errors in browser, click the button, fill the form, test the flow, check responsive, or when needing to visually or functionally verify a web application using Chrome."
---

# Browser Verification via Chrome DevTools Protocol

Testing-grade browser automation using Chrome controlled through the Chrome DevTools Protocol. All interactions use real CDP `Input` domain events (mouse coordinates, key events) — not synthetic JS `.click()`. Zero external dependencies.

## Architecture

All scripts live at `./scripts/` relative to this skill:

- **[chrome-launcher.sh](./scripts/chrome-launcher.sh)** — Start/stop/check Chrome with remote debugging
- **[cdp.mjs](./scripts/cdp.mjs)** — Full CDP client with 50+ commands

No npm install required. Uses Node 22 built-in `WebSocket`.

## Resolving Script Paths

Before running any command, resolve the absolute path to this skill's `scripts/` directory. The scripts are located alongside this SKILL.md file. Use the workspace-relative path `.github/skills/browser-verify/scripts/` to construct the full path in the current workspace root.

For example, if the workspace root is `/path/to/project`, then:
- Chrome launcher: `/path/to/project/.github/skills/browser-verify/scripts/chrome-launcher.sh`
- CDP client: `/path/to/project/.github/skills/browser-verify/scripts/cdp.mjs`

Use these absolute paths in all terminal commands below.

## Workflow

### 1. Start Chrome

```bash
bash "<workspace>/.github/skills/browser-verify/scripts/chrome-launcher.sh" start
# Headless (faster, no window):
bash "<workspace>/.github/skills/browser-verify/scripts/chrome-launcher.sh" start --headless
```

### 2. Run Commands

```bash
node "<workspace>/.github/skills/browser-verify/scripts/cdp.mjs" <command> [args] [--flags]
```

### 3. Stop Chrome

```bash
bash "<workspace>/.github/skills/browser-verify/scripts/chrome-launcher.sh" stop
```

## Command Reference

### Navigation

| Command | Example | Purpose |
|---------|---------|---------|
| `navigate <url>` | `navigate "http://localhost:3000"` | Open URL, wait for load |
| `reload` | `reload` | Reload current page |
| `back` | `back` | Browser back |
| `forward` | `forward` | Browser forward |
| `page-info` | `page-info` | Title, URL, counts, viewport, meta |
| `list-targets` | `list-targets` | List open tabs |
| `new-tab [url]` | `new-tab "http://localhost:3000/about"` | Open new tab |
| `close-tab [id]` | `close-tab` | Close tab |

### Visual

| Command | Example | Purpose |
|---------|---------|---------|
| `screenshot` | `screenshot --output /tmp/page.png` | Full page capture |
| `screenshot --selector <s>` | `screenshot --selector ".card"` | Element capture |
| `screenshot --full` | `screenshot --full --output /tmp/full.png` | Beyond viewport |
| `viewport <w> <h>` | `viewport 375 812` | Set viewport size |
| `pdf` | `pdf --output /tmp/page.pdf` | Save as PDF |
| `emulate <device>` | `emulate mobile` | Device emulation |

Available devices: `mobile`, `iphone`, `ipad`, `tablet`, `android`, `desktop`, `laptop`

### DOM Inspection

| Command | Example | Purpose |
|---------|---------|---------|
| `evaluate <js>` | `evaluate "document.title"` | Run JS in page |
| `get-html [--selector s]` | `get-html --selector "#app"` | Get outerHTML |
| `get-text [--selector s]` | `get-text --selector ".message"` | Get textContent |
| `get-attribute <sel> <attr>` | `get-attribute "a" "href"` | Get attribute |
| `get-value <sel>` | `get-value "input[name=email]"` | Get input value |
| `get-styles <sel>` | `get-styles ".btn" --props "color,fontSize"` | Computed styles |
| `query-all <sel>` | `query-all "li.item"` | Info on all matches |

### Interaction (Real CDP Input Events)

All interactions use real mouse/keyboard events at actual element coordinates.

| Command | Example | Purpose |
|---------|---------|---------|
| `click <sel>` | `click "button.submit"` | Left click |
| `click <sel> --button right` | `click ".menu" --button right` | Right click |
| `click <sel> --count 2` | `click ".item" --count 2` | Double click |
| `hover <sel>` | `hover ".tooltip-trigger"` | Mouse hover |
| `type <sel> <text>` | `type "#email" "test@test.com"` | Type text |
| `type <sel> <text> --clear` | `type "#search" "new query" --clear` | Clear then type |
| `type <sel> <text> --delay 50` | `type "#code" "slow" --delay 50` | Type with delay |
| `press <key>` | `press Enter` | Press key |
| `press <key> --modifiers "ctrl"` | `press a --modifiers "ctrl"` | Key combo (Ctrl+A) |
| `clear <sel>` | `clear "#search"` | Clear input field |
| `focus <sel>` | `focus "#input"` | Focus element |
| `blur <sel>` | `blur "#input"` | Blur element |
| `select <sel> <value>` | `select "#country" "US"` | Select dropdown |
| `check <sel>` | `check "#agree"` | Check checkbox |
| `uncheck <sel>` | `uncheck "#newsletter"` | Uncheck checkbox |
| `scroll <sel>` | `scroll ".footer"` | Scroll to element |
| `scroll --x 0 --y 500` | `scroll --x 0 --y 500` | Scroll to coords |
| `drag <from> <to>` | `drag ".item" ".dropzone"` | Drag and drop |
| `upload <sel> <path>` | `upload "#file" "/tmp/doc.pdf"` | File upload |

Available keys for `press`: `Enter`, `Tab`, `Escape`, `Backspace`, `Delete`, `Space`, `ArrowUp/Down/Left/Right`, `Home`, `End`, `PageUp`, `PageDown`, `F1`-`F12`, or any character.

Modifiers: `ctrl`, `shift`, `alt`, `meta` (comma-separated).

### Waiting

| Command | Example | Purpose |
|---------|---------|---------|
| `wait <sel>` | `wait ".loaded"` | Wait for DOM element |
| `wait-visible <sel>` | `wait-visible ".modal"` | Wait until visible |
| `wait-hidden <sel>` | `wait-hidden ".spinner"` | Wait until hidden |
| `wait-text <text>` | `wait-text "Success"` | Wait for text on page |
| `wait-url <pattern>` | `wait-url "/dashboard"` | Wait for URL change |
| `wait-network-idle` | `wait-network-idle` | Wait for no requests |
| `sleep <ms>` | `sleep 1000` | Fixed delay |

All wait commands accept `--timeout <ms>` (default 10000).

### Debugging

| Command | Example | Purpose |
|---------|---------|---------|
| `get-console` | `get-console` | Console messages |
| `get-network` | `get-network` | Network requests |
| `get-cookies` | `get-cookies` | All cookies |
| `set-cookie <n> <v>` | `set-cookie "token" "abc" --domain localhost` | Set cookie |
| `clear-cookies` | `clear-cookies` | Clear all cookies |

## Common Patterns

### Login Flow Test

```bash
node cdp.mjs navigate "http://localhost:3000/login"
node cdp.mjs type "#email" "user@test.com" --clear
node cdp.mjs type "#password" "pass123" --clear
node cdp.mjs click "button[type=submit]"
node cdp.mjs wait-url "/dashboard" --timeout 5000
node cdp.mjs get-text --selector "h1"
node cdp.mjs screenshot --output /tmp/dashboard.png
```

### Responsive Verification

```bash
node cdp.mjs navigate "http://localhost:3000"
node cdp.mjs emulate mobile
node cdp.mjs screenshot --output /tmp/mobile.png
node cdp.mjs emulate desktop
node cdp.mjs screenshot --output /tmp/desktop.png
```

### Form Validation Test

```bash
node cdp.mjs navigate "http://localhost:3000/register"
node cdp.mjs click "button[type=submit]"
node cdp.mjs wait-visible ".error-message"
node cdp.mjs get-text --selector ".error-message"
node cdp.mjs type "#email" "invalid-email" --clear
node cdp.mjs click "button[type=submit]"
node cdp.mjs get-text --selector ".error-message"
```

### Dropdown + Checkbox

```bash
node cdp.mjs select "#country" "US"
node cdp.mjs get-value "#country"
node cdp.mjs check "#terms"
node cdp.mjs query-all "#terms"
node cdp.mjs screenshot --output /tmp/form.png
```

## Important Notes

- Always start Chrome before running commands; stop when done
- Screenshots saved as PNG — use Read tool to view them
- All `click`/`type`/`hover` use real CDP Input events at element coordinates
- Elements are automatically scrolled into view before interaction
- `type --clear` uses Select All + Delete before typing (framework-safe)
- `query-all` returns element details (tag, text, rect, visibility) for debugging selectors
- Console/network capture is per-session (per `navigate` call)

## Additional Resources

For advanced CDP patterns and troubleshooting:
- **[cdp-commands.md](./references/cdp-commands.md)** — Extended patterns, accessibility checks, advanced evaluate
