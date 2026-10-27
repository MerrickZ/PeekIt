# PeekIt Feature List

| ID | Feature | Status | Issues | Date |
|----|---------|--------|--------|------|
| FEAT-001 | Chinese (zh) UI language + complete i18n (remove hardcoded RU strings) | Completed | kobaltgit/PeekIt#4, #6 | 2026-10-27 |
| FEAT-002 | PDF preview: continuous scroll (infinite scroll paging) | Pending | kobaltgit/PeekIt#5 | 2026-10-27 |
| FEAT-003 | Borderless preview window (content-only mode) | Pending | kobaltgit/PeekIt#3 | 2026-10-27 |

---

## FEAT-001 — Chinese (zh) UI language + complete i18n

**Source issues:** [kobaltgit/PeekIt#4 [FEAT] i18n support](https://github.com/kobaltgit/PeekIt/issues/4), [kobaltgit/PeekIt#6 [BUG] incomplete i18n](https://github.com/kobaltgit/PeekIt/issues/6)

### Requirements
- Add `zh` (Simplified Chinese) as a third interface language besides `ru` and `en`.
- Language selectable in Settings → General → Interface Language (`中文 (ZH)`).
- Eliminate hardcoded Russian strings in components that bypass the i18n layer:
  - `+page.svelte` (group carousel button titles)
  - `GenericPreview.svelte` (folder stats labels)
  - `PluginsTab.svelte` (install/uninstall messages, category labels, confirmations)
  - `PluginHost.svelte` (plugin error title)
  - `plugins.svelte.ts` (store error messages)

### Architecture
- `src/lib/types.ts`: `AppLanguage = 'ru' | 'en' | 'zh'`.
- `src/lib/i18n.ts`: new `zh` translation block with full key coverage + new keys
  (`prev_file`, `next_file`, `folder_contents`, `items`, `files_word`, `folders_word`,
  `plugin_error`, `install_failed`, `uninstall_failed`, `plugin_uninstalled`,
  `confirm_delete`, `delete_q`, `cancel`, `uninstall_plugin`, `disable_plugin`,
  `enable_plugin`, `retry`, `cat_*`).
- Components switch from `lang === 'ru' ? ... : ...` ternaries to `t(key, lang)`.

### Components affected
- `src/lib/types.ts`, `src/lib/i18n.ts`
- `src/routes/+page.svelte`
- `src/lib/components/SettingsModal.svelte`
- `src/lib/components/GenericPreview.svelte`
- `src/lib/components/PluginHost.svelte`
- `src/lib/components/settings/PluginsTab.svelte`
- `src/lib/stores/plugins.svelte.ts`

## FEAT-002 — PDF preview: continuous scroll

**Source issue:** [kobaltgit/PeekIt#5 [FEAT] PDF auto next page when scroll](https://github.com/kobaltgit/PeekIt/issues/5)

### Requirements
- PDF pages rendered in a single vertical scroll container.
- Scrolling down past a page automatically reveals the next page (lazy render on visibility).
- Page indicator follows scroll position; prev/next buttons scroll to the target page.
- Zoom re-renders visible pages at the new scale.

### Architecture
- Rewrite `src/lib/components/PdfPreview.svelte`:
  - Render one `<canvas>` per page inside a scrollable wrapper; placeholder sizing before render.
  - `IntersectionObserver` renders pages entering the viewport (with margin pre-render),
    releases canvases far outside the viewport to bound memory.
  - Scroll handler derives the current page from the element nearest viewport center.

## FEAT-003 — Borderless preview window

**Source issue:** [kobaltgit/PeekIt#3 [FEAT] borderless preview window](https://github.com/kobaltgit/PeekIt/issues/3)

### Requirements
- Optional mode that hides the app chrome (topbar + statusbar + window border), showing only the main preview content.
- Accessible via a toolbar toggle and persisted in settings (`borderlessMode`).
- In borderless mode: hover-revealed floating strip at the top for dragging and essential actions (pin, maximize, settings, close).

### Architecture
- `src/lib/types.ts`: `AppSettings.borderlessMode?: boolean`.
- `src/routes/+page.svelte`: conditional rendering of topbar/statusbar + floating overlay controls.
- `src/lib/components/SettingsModal.svelte`: checkbox in General tab (`borderless_mode` i18n key, RU/EN/ZH).
- `src/lib/i18n.ts`: new keys `borderless_mode`, `borderless_mode_desc`, `exit_borderless`.
