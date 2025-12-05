# Customizable Workspace Themes

## Overview
Enable users to build, install, and share workspace themes that control the editor UI (fonts, color palettes, layout presets). A built-in marketplace will let users browse community themes, preview them instantly, and apply or fork them into their own collections.

## Objectives
- Make it easy to craft personalized editor looks without manual CSS tweaks.
- Allow safe importing/exporting of themes for collaboration and sharing.
- Provide a marketplace experience with ratings, search, and instant previews.
- Keep themes sandboxed so they cannot execute arbitrary code or leak data.

## User Stories
- As a user, I can create a theme by choosing colors, typography, and panel layout presets, then save it for my own use.
- As a user, I can export a theme to share with teammates or publish it to the marketplace with metadata (title, description, tags, cover image).
- As a user, I can browse the marketplace, search/filter themes, and preview a theme before applying it.
- As a user, I can rate or bookmark themes and quickly switch between installed themes from the command palette or settings.
- As a team admin, I can set an organization-wide default theme and restrict publishing to approved members.

## Scope
- **In scope:** theme schema, creation/editor UI, import/export, apply/switch flow, marketplace browsing & discovery, moderation signals (reports, ratings), per-project overrides, persistence to user profile, and migration of existing light/dark toggle into the new system.
- **Out of scope (initially):** programmatic theme hooks for extensions, monetization, and complex animation/transition settings.

## Functional Requirements
### Theme Model
- Theme object includes: id, name, author, version, createdAt/updatedAt, visibility (private/unlisted/public), tags, cover image, rating summary, installs count.
- Style payload covers: base color palette (background, surface, border, text, accent, terminal), syntax highlighting tokens, font stack for UI and code, sizing (line height, border radius, spacing scale), and layout presets (panel sizes, collapsed state, terminal position).
- Must validate schema on save/import; reject unsafe CSS (e.g., `url()` in fonts) and enforce max token counts/size limits.

### Creation & Editing
- New **Theme Builder** in settings with live preview of the editor, terminal, file tree, and chat surfaces.
- Preset templates (e.g., "Midnight", "Solarized", "High Contrast") to start from, plus import from JSON.
- Versioning: edits create a new draft; publishing increments version and preserves changelog.
- Accessibility guardrails: contrast checker with inline warnings before publish/apply.

### Applying & Managing Themes
- Theme switcher accessible from settings and command palette; supports quick preview (temporary apply until confirmed).
- Per-workspace override stored in project settings; user default stored in profile.
- Fallback rules: if a theme is deleted or missing tokens, gracefully fall back to the default palette without breaking the UI.

### Import/Export & Sharing
- Export themes as signed JSON files containing schema version, metadata, and checksum.
- Import flow validates signature, shows preview, and maps missing tokens to defaults.
- Publishing to marketplace requires title, description, tags, cover image, license, and optional source link. Authors can publish as private/unlisted/public.

### Marketplace Experience
- Browse page with search, filters (tags, color style, popularity, recency, verified authors), and sorting.
- Theme detail page with gallery, live sandbox preview, ratings, install count, changelog, and author profile.
- Social signals: users can rate (1–5), bookmark, and report themes. Abuse reports route to moderation queue.
- Server enforces rate limits on publish/update/report actions and requires auth for install/publish/rate.

## Non-Functional Requirements
- Client theme payload under 50KB; lazy-load marketplace previews to avoid blocking editor load.
- Theme application must complete in under 200ms on modern devices and be reversible without reload.
- Offline-friendly: previously installed themes remain usable while offline; marketplace requires connectivity.
- Compatibility: honor existing light/dark theme users by migrating stored preference to default themes.

## Data Model (proposed)
- **Table: themes** — id (uuid), authorId, name, slug, version, visibility, tags (array), coverImage, description, license, ratingAvg, ratingCount, installs, createdAt, updatedAt.
- **Table: theme_tokens** — themeId (fk), section (ui|syntax|layout), key, value.
- **Table: theme_bookmarks** — userId, themeId, createdAt.
- **Table: theme_ratings** — userId, themeId, rating (1–5), review?, createdAt.
- **Table: theme_reports** — id, reporterId, themeId, reason, status, createdAt.
- **Table: theme_installs** — id, userId, themeId, workspaceId?, appliedAt.

## API Surface (sketch)
- `GET /api/themes` — list with search/filter/sort, supports pagination.
- `POST /api/themes` — create/update draft theme; requires payload validation and auth.
- `POST /api/themes/{id}/publish` — publish a version; enforces org permissions if applicable.
- `POST /api/themes/{id}/rate` — add/update rating; rate-limited.
- `POST /api/themes/{id}/bookmark` — toggle bookmark.
- `POST /api/themes/{id}/report` — submit report for moderation.
- `GET /api/themes/{id}` — fetch details + preview payload; respects visibility.
- `GET /api/themes/{id}/install` — serve signed JSON for install/import.

## UX & UI Notes
- Theme Builder uses side-by-side controls: color pickers, font selector, spacing sliders, and layout grid presets (e.g., terminal bottom/right, split ratios).
- Live preview renders a sandbox of editor, file tree, terminal, and chat using the drafted theme payload; revert button restores current theme.
- Marketplace cards show cover image, author, tags, rating, installs, and a "Preview" CTA that opens in a modal with live sandbox.
- Confirm dialogs when applying marketplace themes warn about replacing current customizations; include “apply & keep backup.”

## Migration Plan
1. Introduce theme schema and default light/dark tokens that mirror current styling; add migration that reads `bolt_theme` localStorage and maps to default themes.
2. Implement Theme Builder UI with live preview and JSON import/export.
3. Ship marketplace browsing, install/apply, ratings/bookmarks, and reporting.
4. Add organization defaults and admin controls.

## Risks & Mitigations
- **Malicious payloads:** enforce strict schema validation, sanitize text fields, disallow external font URLs unless whitelisted.
- **Performance regressions:** memoize token application, lazy-load previews, and compress stored payloads.
- **Accessibility regressions:** keep automated contrast checks and require passing scores before publish.
- **Version drift:** include schema version and checksum in exports; reject mismatches with actionable errors.

## Success Metrics
- ≥25% of active users install at least one community theme within 30 days of launch.
- ≥10 curated/verified themes with average rating ≥4.5 within first quarter.
- Theme apply action completes within 200ms on 75th percentile devices.
- Reduction in support tickets related to theme customization/layout tweaks.
