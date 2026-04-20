# tiptap-pagination-plus (performance-hardened fork)

Fork of [RomikMakavana/tiptap-pagination-plus](https://github.com/RomikMakavana/tiptap-pagination-plus)
rewritten for reliability on large documents. Same API surface as
upstream — drop-in replacement via a GitHub URL install. Changes and
fixes are also open as
[PR #31 against upstream](https://github.com/RomikMakavana/tiptap-pagination-plus/pull/31).

## Why this fork

Shipping the extension against ~100-page research narratives exposed
several issues that accumulate with document size:

- Every keystroke ran `Plugin.state.apply`, which called
  `getNewPageCount(view, …)` — reading `getBoundingClientRect` and
  `scrollHeight` on every transaction. ~200 ms of forced reflow per
  keystroke at 100 pages.
- Convergence used one `requestAnimationFrame` per page-count delta
  with no cleanup on unmount — PM would spam
  `view.dom is not available` errors after the editor closed, and
  fast scrolls during convergence revealed unfinished pagination.
- `calculatePageCount` measured content extent via
  `lastElementChild.bottom`, which undercounts when consumers use
  `display: contents` on `<table>` to hoist rows into the editor's
  layout. Silent under-pagination at the document tail.
- Widget page-breaks used `float: left; clear: both`, which any
  `display: table` descendant (including `<tbody>` under the
  extension's own default table CSS) clears — pushing such content
  below the entire widget's float stack. On short-but-table-heavy
  documents this produced runaway page counts in the thousands or
  blank middle pages.
- Extension lifecycle didn't re-converge when content size changed
  after mount (fonts loading, images loading, async content growth).

## Measured results

Numbers from the included puppeteer harness, headless Chrome on an
M-series Mac, synthetic 100-page seed (600 paragraphs + 25 tables):

| Metric                         |    Upstream |   This fork |
| ------------------------------ | ----------: | ----------: |
| Keystroke avg latency          |      200 ms |   **33 ms** |
| Keystroke p95 latency          |      335 ms |   **33 ms** |
| Forced reflow (DevTools trace) |      316 ms |       94 ms |
| LCP                            |     2079 ms |      307 ms |
| First convergence              |      728 ms |      ~200 ms |

Correctness fixes confirmed on customer-supplied reproducers
(anonymised and bundled under `perf/`):

| Document shape                                 | Upstream             | This fork                    |
| ---------------------------------------------- | -------------------- | ---------------------------- |
| ~28 pages, many distributed tables             | tail under-paginated | 32 pages, 0 blank, 0 overflow |
| Short doc (~10 kpx) with one early `<tbody>`   | runaway to 1000+ pg  | 12 pages, 0 blank             |

## Install

Install from GitHub at a release tag. `prepare: tsc` builds `dist/`
on install, so no separate build step is needed.

```bash
npm install github:akamick86/tiptap-pagination-plus#v1.0.0
```

Works with both TipTap v2 and v3 (peer deps cover both).

## Usage

```typescript
import { Editor } from '@tiptap/core'
import StarterKit from '@tiptap/starter-kit'
import { PaginationPlus, PAGE_SIZES } from 'tiptap-pagination-plus'

const editor = new Editor({
  extensions: [
    StarterKit,
    PaginationPlus.configure({
      pageHeight: 800,
      pageWidth: 789,
      marginTop: 20,
      marginBottom: 20,
      marginLeft: 50,
      marginRight: 50,
      pageGap: 50,
      contentMarginTop: 10,
      contentMarginBottom: 10,
      footerRight: '{page}',
      footerLeft: '',
      headerRight: '',
      headerLeft: '',
      customHeader: {},
      customFooter: {},
    }),
  ],
  content: '…',
})
```

Same commands as upstream: `updatePageHeight`, `updatePageWidth`,
`updatePageGap`, `updateMargins`, `updateContentMargins`,
`updateHeaderContent`, `updateFooterContent`,
`updatePageBreakBackground`, `updatePageSize`.

## Development

```bash
# typecheck + build dist
npm run build

# interactive reproduction (port 7777)
npm run perf:serve
# → http://127.0.0.1:7777/playground.html?paragraphs=600&tables=25

# automated regression test (headless Chrome, asserts thresholds)
npm run test:perf
```

The `test:perf` harness fails the run if convergence exceeds
`CONVERGENCE_MAX_MS` (default 4000) or keystroke p95 exceeds
`KEYSTROKE_P95_MAX_MS` (default 250). Both thresholds are
env-overridable for slower CI boxes. See `perf/README.md` for details.

## License

[MIT](LICENSE). Original extension Copyright (c) 2024 Romik Makavana;
fork additions Copyright (c) 2024 Mikhail Abramchyk.
