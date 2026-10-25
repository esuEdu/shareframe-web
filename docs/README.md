# shareframe-web docs

Design and stack decisions for the React frontend. **Design documents, not a
description of shipped code** — the repository currently contains no
implementation.

## Contents

| Document | Covers |
| --- | --- |
| [`design-decisions.md`](design-decisions.md) | Layout, theme, and the UI rules that carry cost consequences |
| [`frontend-stack.md`](frontend-stack.md) | Styling, components, state, LiveKit SDK usage, auth, testing |

**Visual mockup:** https://claude.ai/artifact/8MaVfQaQn63gRr99Hhs3xi — four
artboards (room, watch picker, ad break, join), dark-first, pt-BR. Private
unless shared from the page's share menu.

## The decisions

| | |
| --- | --- |
| Layout | **Stage + rail** — one large selected screen, compact sharing rail. No tile grid. |
| Theme | **Dark-first**, light supported, one token set |
| Rail & picker | **Static poster frames only.** Never live video. |
| Sharing | Always available to everyone. No queue or host approval. |
| Styling | Tailwind v4, tokens in `@theme` as CSS variables |
| Components | Radix primitives copied in (shadcn style) — own the design system |
| State | Zustand; separate room store and UI store |
| LiveKit | Hooks only, `autoSubscribe: false`. **Never the prebuilt layouts.** |
| Auth | Plain redirect to the backend. No client-side OAuth library. |
| Tests | Vitest + RTL, Playwright for the room flow |

## Why the layout is not a grid

Worth stating up front, because it contradicts the repo README's current scope
list and the reflex of anyone who has built a meeting app.

**There are no webcams in this product.** ShareFrame shares screens and optional
system audio. A participant grid would be a grid of name labels with nothing to
show.

**Viewers subscribe to one screen at a time**, capped server-side. A grid of
equal tiles promises simultaneous viewing the system deliberately does not
provide — and filling one would mean breaking the cost model.

So: one stage, one rail, and a roster of avatars. See decision 1 in
[`design-decisions.md`](design-decisions.md).

## The two ways to make this expensive

Both are one-line changes that look like features:

1. **Live thumbnails in the rail or picker.** Ten viewers previewing ten screens
   is ~13.5 GB/hour — more than the screens anyone is actually watching.
2. **Using `@livekit/components-react`'s prebuilt layouts.** They subscribe to
   every track to fill a grid, which is precisely the case the backend design
   exists to avoid.

Guard both in review. The backend caps subscriptions server-side, so neither
mistake will break the app — it will just quietly cost nine times as much.

## Related

- [`shareframe-api`](https://github.com/esuEdu/shareframe-api) — Go backend, OAuth2, room permissions and LiveKit tokens
- [`shareframe-api/docs`](https://github.com/esuEdu/shareframe-api/tree/main/docs) — product decisions, cost model and hosting
