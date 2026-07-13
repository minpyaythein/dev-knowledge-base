# Dark mode without the flash — apply the theme in an inline head script

A theme stored in localStorage must be applied by a blocking inline `<script>` in `<head>`, before the body renders — applying it from React/JS bundles paints one wrong-theme frame first (the "flash").

## Why it matters

The app bundle loads asynchronously; by the time a `useEffect` (or any bundle code) adds the `dark` class, the browser has already painted the page in the default theme. Users on dark mode see a white flash on every load. The fix must run before first paint, which means inline in the HTML, not in the bundle.

## How it works

In `index.html`, right in `<head>`:

```html
<script>
  // Apply the stored (or system) theme before first paint — no flash.
  try {
    var t = localStorage.getItem('my-app-theme');
    if (t === 'dark' || (t !== 'light' && matchMedia('(prefers-color-scheme: dark)').matches)) {
      document.documentElement.classList.add('dark');
    }
  } catch (e) {}
</script>
```

The pieces that matter:

- **Three-state logic**: stored `'dark'` → dark; stored `'light'` → light; *nothing stored* → follow the OS via `prefers-color-scheme`. Users who never touched the toggle get their system theme; an explicit choice wins and persists.
- **Class on `document.documentElement`** (`<html>`), because `<body>` doesn't exist yet at this point in parsing. CSS keys off it: `:root.dark { --bg: …; }` with theme values as custom properties.
- **`try/catch`**, because `localStorage` throws in some privacy modes — a broken theme script must never break the page.
- The runtime toggle (React state, etc.) then only needs to flip the same class and write the same localStorage key; the inline script and the app agree on one source of truth.

## Gotchas

Keep the inline script tiny and dependency-free — it runs before everything, blocks parsing, and can't import anything. Theme *application* lives here; theme *UI* stays in the app bundle.
