# CDP Commands — Extended Reference

## Advanced Evaluate Patterns

### Check for JavaScript Errors

```bash
node cdp.mjs evaluate "(() => {
  const errors = [];
  window.addEventListener('error', e => errors.push(e.message));
  return JSON.stringify(errors);
})()"
```

### Get Computed Styles

```bash
node cdp.mjs evaluate "(() => {
  const el = document.querySelector('.target');
  const styles = getComputedStyle(el);
  return JSON.stringify({
    color: styles.color,
    fontSize: styles.fontSize,
    display: styles.display,
    position: styles.position,
    margin: styles.margin,
    padding: styles.padding,
  });
})()"
```

### Check Accessibility

```bash
node cdp.mjs evaluate "(() => {
  const issues = [];
  document.querySelectorAll('img').forEach(img => {
    if (!img.alt) issues.push({ tag: 'img', src: img.src, issue: 'missing alt' });
  });
  document.querySelectorAll('a').forEach(a => {
    if (!a.textContent.trim() && !a.getAttribute('aria-label'))
      issues.push({ tag: 'a', href: a.href, issue: 'empty link text' });
  });
  document.querySelectorAll('input, select, textarea').forEach(input => {
    const id = input.id;
    if (id && !document.querySelector('label[for=\"' + id + '\"]'))
      issues.push({ tag: input.tagName, id, issue: 'no associated label' });
  });
  return JSON.stringify(issues);
})()"
```

### Get All Links

```bash
node cdp.mjs evaluate "JSON.stringify(Array.from(document.links).map(a => ({ href: a.href, text: a.textContent.trim().slice(0, 80) })))"
```

### Check Responsive Layout

```bash
# Resize viewport then screenshot
node cdp.mjs evaluate "(() => {
  // Get current layout info at various breakpoints
  const el = document.querySelector('.container');
  const rect = el?.getBoundingClientRect();
  return JSON.stringify({
    viewport: { width: window.innerWidth, height: window.innerHeight },
    container: rect ? { width: rect.width, height: rect.height } : null,
    mediaQueries: {
      isMobile: window.matchMedia('(max-width: 768px)').matches,
      isTablet: window.matchMedia('(min-width: 769px) and (max-width: 1024px)').matches,
      isDesktop: window.matchMedia('(min-width: 1025px)').matches,
    }
  });
})()"
```

### Check React Component State (if React DevTools hook available)

```bash
node cdp.mjs evaluate "(() => {
  const root = document.getElementById('root');
  const fiberKey = Object.keys(root || {}).find(k => k.startsWith('__reactFiber'));
  if (!fiberKey) return JSON.stringify({ react: false });
  return JSON.stringify({ react: true, fiber: 'found' });
})()"
```

## Multi-Tab Verification

### Open Multiple Pages and Compare

```bash
# Open two tabs
node cdp.mjs new-tab "http://localhost:3000/page-a"
node cdp.mjs new-tab "http://localhost:3000/page-b"

# List to get target IDs
node cdp.mjs list-targets

# Screenshot each (use --target with the target ID from list-targets)
node cdp.mjs screenshot --target <id-a> --output /tmp/page-a.png
node cdp.mjs screenshot --target <id-b> --output /tmp/page-b.png
```

## Network Request Analysis

After navigating to a page, use `get-network` to capture:
- All XHR/fetch requests and their status codes
- Resource loading (CSS, JS, images)
- Failed requests (status >= 400)
- API response checking

```bash
node cdp.mjs navigate "http://localhost:3000"
# Interact with the page to trigger API calls...
node cdp.mjs click ".load-data-btn"
node cdp.mjs wait ".data-loaded"
node cdp.mjs get-network
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `CDP_PORT` | `9222` | Chrome debugging port |
| `CDP_HOST` | `127.0.0.1` | Chrome debugging host |

## Troubleshooting

### Chrome won't start
- Check if another Chrome debug instance is running: `lsof -i :9222`
- Kill stale processes: `bash chrome-launcher.sh stop`
- Check Chrome binary path detection in `chrome-launcher.sh`

### WebSocket connection failed
- Ensure Chrome is running: `bash chrome-launcher.sh status`
- Verify the port: `curl http://127.0.0.1:9222/json/version`

### Screenshot is blank
- Page may not have loaded fully; add a `wait` for a key element before screenshot
- Check if the page requires user interaction or auth

### Element not found
- Verify selector with `evaluate`: `document.querySelector('.my-el')`
- Element may be in a shadow DOM — use `evaluate` with `shadowRoot` traversal
- Element may be in an iframe — not directly accessible via top-level CDP
