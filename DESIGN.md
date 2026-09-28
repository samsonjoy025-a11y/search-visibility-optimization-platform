# Design system

**Product:** Search Visibility Optimization Platform  
**Visual preview:** [`design.html`](./design.html)  
**Architecture:** [`ARCHITECTURE.md`](./ARCHITECTURE.md)

Dense, evidence-first dashboard language for SEO agencies. Desktop-first. Default light theme. Optional dark later.

---

## Principles

- Never stop at a raw issue count. Always: category → priority → next action → task.
- Every recommendation shows **Evidence, Reason, Confidence, Action, Measurement**.
- Label **Observed evidence** vs **Model interpretation**.
- Do not promise rankings. Report observed change, not causation.
- Color is not the only signal for rank movement (icons + text).

---

## Type

| Role | Font | Size / weight |
|---|---|---|
| UI + body | IBM Plex Sans | 15 / 400 |
| Display | IBM Plex Sans | 36 / 700, tracking −0.03em |
| Title | IBM Plex Sans | 22 / 600 |
| Caption | IBM Plex Sans | 13 / 400 |
| Data / URLs / formulas | IBM Plex Mono | 13 / 400 |

Load: [Google Fonts IBM Plex Sans + Mono](https://fonts.google.com/specimen/IBM+Plex+Sans).

---

## Color tokens

| Token | Hex | Use |
|---|---|---|
| Ink | `#12161C` | Primary text |
| Secondary | `#3D4754` | Supporting text |
| Muted | `#6B7684` | Captions, labels |
| Canvas | `#F4F6F8` | Page background |
| Surface | `#FFFFFF` | Cards |
| Surface muted | `#EEF1F4` | Nested panels |
| Line | `#D8DEE6` | Borders |
| Brand | `#0F6E56` | Primary actions, eyebrow |
| Brand hover | `#0C5A46` | Primary hover |
| Brand soft | `#E6F4EF` | Focus ring, ghost hover |
| Accent | `#1D4ED8` | Links, secondary emphasis |
| Critical | `#B42318` | Blocking index/discovery issues |
| High | `#C2410C` | High-impact issues |
| Medium | `#A16207` | Optimization opportunities |
| Low | `#475569` | Minor issues |
| Healthy | `#15803D` | Project health |
| Attention | `#CA8A04` | Needs review |
| Positive / up | `#047857` | Ranking or metric improvement |
| Decline / down | `#BE123C` | Ranking or metric drop |
| Observed | `#0F766E` | Evidence from crawl/GSC |
| Interpreted | `#6D28D9` | LLM or inferred labels |

Radius: `8px`. Cards: `12px`. Shadow: `0 1px 2px rgba(18,22,28,.06), 0 8px 24px rgba(18,22,28,.04)`.

---

## Buttons

| Variant | Style |
|---|---|
| Primary | Brand fill, white text — Start optimization, Connect Search Console |
| Secondary | White, line border — Create task, View evidence |
| Ghost | Transparent, brand text — Snooze |
| Danger | Critical fill — Disconnect |
| Small / large | 13px compact tables / 15px onboarding |
| Disabled | 45% opacity — Running crawl… |

---

## Inputs

- 14px IBM Plex Sans, 9px/12px padding, 8px radius, line border.
- Focus: brand border + brand-soft ring.
- Error: critical border + helper text in critical.
- Hint: 12px muted under the field.

---

## Product components to implement in the app

App shell (workspace + project switcher), data table, metric card + sparkline, evidence panel, recommendation card, Next Best Action hero, severity badges, scoring breakdown, task pipeline, change-log timeline, before/after metrics, integration OAuth cards, empty/loading/partial/error/rate-limit states.

---

## Implementation

- **CSS:** variables as in `design.html`, then Tailwind + shadcn/ui in Next.js.
- **A11y:** WCAG AA, keyboard tables, do not rely on color alone.
- **Breakpoints:** desktop dashboards; mobile for status, NBA, and task actions only.
