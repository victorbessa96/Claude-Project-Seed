# UI/UX Patterns

Reusable component patterns and interaction design principles. These are about behavior and structure, not visual design — adapt them to your project's aesthetic.

---

## Navigation Patterns

### Sidebar Navigation
Best for: tool-heavy applications with 4+ sections, desktop-first workflows.

- Fixed position, always visible (collapsible to icon-only mode)
- Active section indicated visually and via `aria-current="page"`
- Collapse/expand state persisted in localStorage
- Auto-collapse at tablet breakpoint (~1024px)
- Icons + labels when expanded; icons only when collapsed
- Inline SVG icons preferred over icon fonts or CDN dependencies

**Why:** Sidebars provide persistent navigation without consuming content space. Users of data-heavy tools switch between sections frequently — the sidebar makes all sections one click away.

**When to use something else:**
- **Tabs** — for 2-4 sections where content switching is the primary interaction
- **Top navbar** — for content-focused sites (blogs, docs) where navigation is secondary
- **Breadcrumbs** — for deep hierarchical navigation (file managers, admin panels)

### Hash-Based Routing (no-build SPAs)
- Use URL hashes for view navigation (`#/dashboard`, `#/settings`)
- Map hashes to render functions in a route dictionary
- Listen to `hashchange` events for navigation
- Update active nav indicators on every route change
- Support a default route (redirect unknown hashes to home)

### View Transitions
- Add fade-in or slide-in animations when switching views
- Force reflow between removing and adding animation classes (`offsetWidth` trick)
- Keep transitions fast (200-300ms) — they communicate state change, not entertain

---

## Loading States

Every view that loads data asynchronously needs a loading state. The user should never see a blank screen and wonder if the app is broken.

### Skeleton Placeholders
Best for: content areas where the layout is known but data isn't loaded yet.

- Render placeholder shapes that mimic the final content structure
- Use CSS animation (pulse/shimmer) to indicate loading
- Replace skeletons with real content once data arrives

**Why:** Skeletons feel faster than spinners because they communicate structure. Users perceive the page as "almost loaded" rather than "loading."

### Spinners
Best for: small areas (buttons, inline indicators) or when the final layout is unknown.

- Show after a brief delay (~200ms) to avoid flickering on fast loads
- Keep them small and unobtrusive
- Disable the trigger button while loading to prevent double-submission

### Progress Bars / Indicators
Best for: long-running operations (imports, exports, multi-step processes).

- Show step-by-step progress when possible ("Step 2/5: Processing articles")
- Use server-sent events (SSE) for real-time updates on long operations
- Include a cancel/abort mechanism for user-triggered operations

---

## Notifications

### Toast System
Non-blocking notifications that appear briefly and auto-dismiss.

- **Types:** success (green), error (red), warning (amber), info (blue)
- **Behavior:** slide in from a corner, auto-dismiss after 3-5 seconds, dismissable on click
- **Progress bar:** visual countdown to auto-dismiss
- **Stacking:** multiple toasts stack vertically without overlapping
- **Accessibility:** container has `aria-live="polite"` for screen reader announcements
- **Security:** escape HTML in toast messages to prevent XSS

**Why:** Toasts give feedback without interrupting workflow. They confirm actions ("Article saved"), report errors ("Fetch failed: timeout"), and surface information ("3 new articles found") without modal dialogs.

**When to use modals instead:** Destructive actions (delete confirmation), complex inputs (multi-field forms), or situations requiring the user's full attention before proceeding.

> **CyberPulse example:** Toast notifications with slide-in animation, progress bar countdown (5s), and click-to-dismiss. HTML content is escaped via `textContent` → `innerHTML` pattern to prevent XSS.

---

## Tables & Data Display

### Sortable Columns
- Column headers are clickable to sort
- Click once for ascending, again for descending
- Visual indicator (arrow icon) shows current sort column and direction
- Sort state persisted in localStorage
- Backend accepts `sort_by` and `sort_dir` with whitelist validation

### Pagination
- Show total count: "247 articles — Page 3/13"
- Previous / Next buttons (disable when at first/last page)
- Reasonable page size (25-50 items)
- Page state persisted in URL or localStorage

### Filters
- Multi-select filters for categorical data (source, category, status)
- Update results on filter change (with debounce for text inputs)
- Show active filter count as a badge
- "Clear all" button that resets all filters and persisted state
- Filter state persisted in localStorage for session continuity

### Empty States
- When no data exists yet: friendly message with guidance ("No articles yet. Run a fetch to get started.")
- When filters return no results: "No matching articles. Try adjusting your filters."
- Never show a blank table with just column headers

> **CyberPulse example:** Articles table has sortable columns, source/category/relevance filters persisted in localStorage (`articles-view-state` key), pagination with total counts, and a "Clear" button that wipes persisted state.

---

## Forms & Input

### Validation Feedback
- Validate on blur (when the user leaves a field), not on every keystroke
- Show error messages next to the relevant field, not in a separate area
- Use color + icon + text for error states (never rely on color alone — accessibility)
- Clear error state when the user starts correcting the input

### Submit States
- Disable the submit button while the request is in flight
- Show a loading indicator on the button itself
- On success: toast notification + redirect or content update
- On error: re-enable the button, show the error, preserve the user's input

### Field Grouping
- Group related fields visually (border, background, or spacing)
- Use fieldset/legend for accessibility when appropriate
- Required fields marked consistently (asterisk, "required" label)
- Sensible defaults for optional fields — minimize what the user must fill in

### Advanced Validation Patterns

**Async validation** — Some fields need server-side checks (is this username taken? is this URL reachable?). Validate after the user finishes typing (debounce 300-500ms), show a loading indicator on the field, and display the result inline. Don't block form submission on slow async checks — validate optimistically and handle server-side rejection on submit.

**Dependent fields** — When one field's value determines what other fields are available (e.g., selecting "RSS" as source type reveals a URL field). Show/hide dependent fields dynamically. Clear dependent values when the parent changes. Ensure hidden fields don't submit values.

**Multi-field validation** — Some rules span multiple fields (password confirmation, date ranges where start must precede end). Validate these on submit or when the last related field loses focus — not on every keystroke. Show the error next to the most relevant field, not in a banner.

---

## Modal & Dialog Patterns

### When to Use Modals
- **Confirmation of destructive actions** — "Delete 15 articles? This cannot be undone."
- **Focused input** — A form that needs the user's full attention (settings that affect the whole system)
- **Preview/detail** — Showing expanded content without navigating away from a list view

### When NOT to Use Modals
- Simple acknowledgements ("Saved!") — use a toast instead
- Content that needs scrolling or complex interaction — use a full page
- Frequent actions — modal fatigue makes users dismiss without reading

### Modal Behavior
- **Focus trap** — Tab cycles through modal elements only; focus never escapes to the background
- **Escape to close** — Always. No exceptions.
- **Backdrop click** — Closes the modal for informational dialogs. For forms with unsaved data, either ignore backdrop clicks or warn before closing.
- **Return focus** — When the modal closes, focus returns to the element that opened it
- **No nested modals** — If you need a modal inside a modal, your UX needs a redesign

### Sizing
- Content determines size, up to a reasonable maximum (e.g., 600px width for forms, 800px for previews)
- On mobile, modals expand to near-full-screen
- Scrollable content inside the modal, not scrolling the modal itself

---

## Search Patterns

### Search Input
- Prominent placement — top of content area or in the header
- Debounce input (300ms) before triggering search
- Show a loading indicator during search
- Clear button (×) when search has content
- Preserve search term across navigation (if the search applies globally)

### Results Display
- Highlight matching terms in results
- Show the most relevant information first (title, then context snippet)
- For filtered lists (searching within a table), update the list in place
- For global search, show results in a dropdown or dedicated results view

### Empty and Error States
- **No results:** "No articles matching 'ransomware attack'. Try different keywords." — be specific about what was searched
- **Too many results:** Suggest narrowing with filters rather than paginating through hundreds
- **Search error:** Degrade gracefully — show the last good results or the unfiltered view, with a toast explaining the error

> **CyberPulse example:** Article table filter supports text search across titles with 300ms debounce, combined with source and category filters. Active search terms are preserved in localStorage with other filter state.

---

## Bulk Actions

### Multi-Select
- Checkbox on each row for individual selection
- "Select all" checkbox in the table header — selects visible items, not all pages
- Selection count visible: "12 articles selected"
- Selection persists across pagination only if explicitly designed (and clearly communicated to the user)

### Action Bar
- Appears when items are selected (floating bar or inline above the table)
- Shows available actions: Delete, Export, Change Category, etc.
- Destructive bulk actions always require confirmation with the count: "Delete 12 articles?"
- Non-destructive actions (export, tag) can proceed without confirmation

### Feedback
- Progress indication for bulk operations that take time
- Summary toast on completion: "12 articles deleted" or "10 exported, 2 failed"
- If some items fail, report the failures specifically (which items, why)

---

## Responsive Design

### Breakpoint Strategy

| Breakpoint | Behavior |
|-----------|----------|
| ≥1280px | Full layout, all panels visible |
| ≥1024px | Sidebar auto-collapses; content takes full width |
| <1024px | Sidebar hidden or overlay; hamburger menu |
| <640px | Mobile stack layout; single column; larger touch targets |

**Why:** Desktop-first is appropriate for data-heavy tools where most users are on desktop. Mobile support ensures the app is usable on tablets and phones, even if it's not the primary use case.

**In practice:**
- Use CSS media queries, not JavaScript, for layout changes
- Test at each breakpoint during development
- Collapsible panels (sidebar, filter drawers) preserve content space on smaller screens
- Touch targets should be at least 44x44px on mobile

---

## Accessibility

Accessibility is a requirement, not a feature. These are minimum expectations.

### ARIA Attributes
- Navigation: `aria-current="page"` on the active link
- Notifications: `aria-live="polite"` on the toast container
- Expandable elements: `aria-expanded` on toggle buttons
- Icon buttons: `aria-label` describing the action (not the icon)
- Modals/dialogs: `aria-modal`, `aria-labelledby`, focus trap

### Keyboard Navigation
- All interactive elements reachable via Tab
- Escape closes modals and dropdowns
- Enter activates buttons and links
- Arrow keys navigate within lists and menus

### Color & Contrast
- Never use color as the only indicator (add icons or text)
- Minimum 4.5:1 contrast ratio for body text
- Minimum 3:1 for large text and UI components

---

## Internationalization (i18n)

Even if your app launches in one language, externalizing strings from the start makes adding languages trivial later.

### Architecture
- All UI strings in a single translations module (not scattered across view files)
- `t("key")` helper function for lookup, with fallback to English, then to key itself
- Interpolation support: `t("articles_count", count)` → "247 articles"
- DOM localization via `data-i18n` attributes on static elements, updated by a `localizeDOM()` function

### Language Switching
- Language preference stored in settings (server-side) or localStorage (client-side)
- Switching language re-renders all visible text without a page reload
- Content generation (LLM, templates) respects the language setting independently of UI language

**Why:** Retrofitting i18n is painful — it touches every file. Starting with externalized strings costs almost nothing and makes the codebase more organized (all text in one place).

> **CyberPulse example:** `i18n.js` contains all UI strings in EN and PT. `t(key, ...args)` with `{0}`/`{1}` interpolation. Sidebar uses `data-i18n` attributes. Language loaded from `default_language` setting on init. LLM prompts also have bilingual variants.

---

## Theming

### CSS Custom Properties (Design Tokens)
- Define colors, spacing, typography, and component dimensions as CSS variables
- All components reference tokens, never hardcoded values
- Tokens enable theme switching (dark/light) with a single class change on the root element

### Dark Mode Considerations
- Dark backgrounds with light text (not just inverted colors)
- Reduce contrast slightly — pure white on pure black is harsh
- Use color accents sparingly for CTAs and active states
- Semantic colors (success, error, warning) need muted variants for dark backgrounds

---

## State Persistence

### What to Persist in localStorage
- UI preferences: sidebar collapsed/expanded, sort column/direction, active filters
- Non-sensitive, non-critical state that enhances continuity across page loads
- Always provide a "reset to defaults" mechanism

### What NOT to Persist in localStorage
- Sensitive data (tokens, API keys, personal information)
- Data that the server is the source of truth for
- Large datasets (localStorage has ~5MB limit)

---

## First-Run Experience

The first time a user opens the app, guide them to essential setup.

- Detect missing configuration (API key, required settings)
- Redirect to settings/setup screen automatically
- Show a setup banner or wizard explaining what's needed
- After setup, redirect to the main view

**Why:** An app that shows a blank dashboard with cryptic errors on first run loses users immediately. A smooth first-run experience builds confidence.

> **CyberPulse example:** On init, the app checks for an empty `openrouter_api_key`. If missing, it redirects to `#/settings` and shows a setup banner.

---

## Related Documents

- [03-architecture](03-architecture.md) — SPA patterns and frontend architecture
- [06-security](06-security.md) — XSS prevention in UI, input validation
- [08-performance](08-performance.md) — Lazy loading, bundle size
