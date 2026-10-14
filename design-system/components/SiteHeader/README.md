# SiteHeader

The happycynic.com masthead: the full logo centred above a short, pipe-separated nav on `surface-raised` with `shadow-md`.

**Markup:** `<header class="hc-header">` holding `.hc-header__logo` (two images: `.hc-logo--full` and `.hc-logo--face`) and `<nav class="hc-nav">` with links separated by `<span class="hc-nav__sep" aria-hidden="true">|</span>`. Mark the current page with `aria-current="page"`.

- Keep the nav to three to five Title Case items.
- Both logo images are required: the dark theme hides the full logo (its wordmark is black) and shows the face alone.
- Product sites use their own header; this one is for Happy Cynic properties.
