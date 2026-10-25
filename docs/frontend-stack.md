# Frontend stack

Decided 24 October 2026. React · TypeScript · Vite were already set; these are
the remaining choices.

## Styling: Tailwind v4 with CSS-variable tokens

**Decision: Tailwind CSS v4, with the design system's tokens declared in
`@theme` so they compile to CSS variables.**

```css
@import "tailwindcss";

@theme {
  --color-stage:      oklch(0.17 0.01 260);
  --color-surface:    oklch(0.22 0.012 260);
  --color-accent:     oklch(0.70 0.17 250);
  --color-live:       oklch(0.72 0.19 20);
  --radius-panel:     0.75rem;
}
```

Why this over the alternatives:

- **vs. CSS Modules** — Tailwind v4's `@theme` gives one declaration site that
  produces both utility classes and runtime CSS variables. With CSS Modules the
  token layer has to be hand-built and kept in sync.
- **vs. styled-components / Emotion** — both carry runtime cost. This app's
  render path sits next to a live media pipeline on a 2 OCPU server's worth of
  users; zero-runtime styling is the cheaper default.
- **vs. vanilla-extract** — genuinely good for design systems and type-safe, but
  a smaller ecosystem and more build wiring for no benefit this product needs.

Theming is a variable swap, so dark-first with a supported light theme (decision
4 in [`design-decisions.md`](design-decisions.md)) is one token block
redeclared under `:root[data-theme="light"]`, not a second stylesheet.

## Components: Radix primitives, copied in

**Decision: own the components. Use Radix UI primitives for behaviour, copied
into the repo in the shadcn/ui style — not installed as a styled component
library.**

The repo README commits to a design system with "tokens, primitives and shared
components", which rules out MUI or Chakra: adopting someone else's visual
language and then fighting it is the worst of both. But hand-writing an
accessible dialog, dropdown, tooltip and slider is weeks of work with subtle
focus-management bugs.

Radix gives unstyled, accessible behaviour; copying the thin wrappers into
`src/components/ui/` means the design system is yours and there is no upstream
styling to override. The accessibility work is borrowed, the design is not.

## State: Zustand for room state

**Decision: Zustand. Not Redux, not Context for anything that changes often.**

Room state is small but updates at media frequency — participants joining,
tracks publishing, the selected stage changing. Context re-renders every
consumer on every change, which is wrong for a view containing a live video
element. Redux's ceremony buys nothing at this size.

Keep two stores separated:

- **Room store** — participants, who is publishing, the selected stage track.
  Derived from LiveKit events.
- **UI store** — picker open, theme, panel collapsed. Persisted to
  `localStorage`.

## LiveKit: hooks only, never the prebuilt UI

**Decision: use `livekit-client` and the hooks from
`@livekit/components-react`. Do not use its prebuilt layout components
(`VideoConference`, `GridLayout` and friends).**

This is the most important line in this document. Those prebuilt components
exist to render a conference grid, and they subscribe to every available track
to fill it. Dropping one in would be a one-line change that re-creates the
81 GB/hour case the entire backend design avoids, and it would look like
progress while doing it.

Required connection options:

```ts
const room = new Room({
  dynacast: true,        // pause published tracks nobody is watching
  adaptiveStream: true,  // match delivered quality to element size
});

await room.connect(url, token, {
  autoSubscribe: false,  // the whole point: we choose
});
```

Then subscribe explicitly on stage selection, unsubscribing the outgoing track
first, per decision 3:

```ts
outgoing?.setSubscribed(false);
incoming.setSubscribed(true);
```

`autoSubscribe: false` is the client-side half of the one-stream cap. The
server-side half is LiveKit participant permissions, set by the API. Both exist
because the client half is what makes it fast and the server half is what makes
it true. Verify these option names against the current SDK version at
implementation time.

## Auth: redirect to the backend, no SDK

**Decision: no Google SDK, no `@react-oauth/google`. The login button is a plain
link to `GET /v1/auth/google/start`.**

The backend runs the OAuth2 flow as a confidential client and sets an HttpOnly
session cookie. The SPA never holds a token, so there is nothing for a
client-side auth library to manage. Session state comes from
`GET /v1/auth/me`; sign-out is `DELETE /v1/auth/session`.

Render the provider buttons from `GET /v1/auth/providers` rather than hardcoding
Google, so enabling Facebook later needs no frontend release.

All API calls send `credentials: "include"`.

## Testing

**Decision: Vitest + React Testing Library for units, Playwright for the room
flow.**

The tests that earn their keep here are the ones guarding the cost model, since
every other bug is merely visible:

- switching stages leaves exactly **one** subscribed track
- the rail and picker mount **zero** `<video>` elements
- a server-initiated unsubscribe renders the overlay, not an error
- the Share button is enabled regardless of how many others are sharing

Playwright can assert subscription counts through the LiveKit room object, which
makes the first two mechanically checkable rather than review-dependent.

## Not deciding yet

- **Routing** — React Router is the default assumption; the app is roughly three
  routes and nothing hinges on it.
- **Data fetching** — plain `fetch` in a thin client module until there is
  enough server state to justify TanStack Query.
- **i18n** — Brazil-only means pt-BR first. Worth structuring strings behind a
  lookup from the start; not worth a library yet.
