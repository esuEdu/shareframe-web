# Design decisions

Decided 24 October 2026. These follow from the backend product decisions in
[`shareframe-api/docs`](https://github.com/esuEdu/shareframe-api/tree/main/docs);
where a UI choice has a cost consequence, that is called out.

## 1. Stage + rail, not a participant grid

**Decision: one large stage showing the selected screen, with a compact rail of
who else is sharing. No tile grid.**

Two facts force this, and neither is a style preference:

**There are no webcams.** ShareFrame shares screens and optional system audio.
No camera tracks exist anywhere in the product. A "participant grid" would be a
grid of name labels — it has nothing to display. The repo README's mention of a
participant grid is a leftover from a video-conferencing mental model and is
wrong for this product.

**Each viewer subscribes to one screen at a time.** The backend caps
subscriptions at 1, enforced server-side. A grid of equal tiles visually
promises simultaneous viewing that the system does not provide, and building one
would mean either lying about the layout or breaking the cost model to fill it.

The layout that matches the product:

```
┌────────────────────────────────────────────────────────┐
│  room name · 7 people                      [ Share ]   │ ← chrome
├──────────────────────────────────────────┬─────────────┤
│                                          │ SHARING (3) │
│                                          │ ┌─────────┐ │
│                                          │ │ ▣ Ana   │ │ ← rail:
│            THE STAGE                     │ │ (poster)│ │   poster
│      the one screen you chose            │ └─────────┘ │   frames,
│                                          │ ┌─────────┐ │   not live
│                                          │ │ ▣ Bruno │ │   video
│                                          │ └─────────┘ │
│                                          ├─────────────┤
│                                          │ IN ROOM (7) │
│                                          │ ◍ ◍ ◍ ◍ ◍   │ ← avatars
└──────────────────────────────────────────┴─────────────┘
```

The stage gets the pixels because the shared screen is the product. Everything
else is navigation.

## 2. The rail is a cost boundary, not a component

**Decision: the rail and the screen picker show static poster frames. Never live
video.**

This is the single most expensive mistake available in this codebase. A rail of
live thumbnails for ten shared screens is:

```
10 viewers × 10 thumbnails × 300 kbps = 30 Mbps = 13.5 GB/hour
```

— more downstream traffic than the full-quality screens people are actually
watching, and it silently converts the room from ~9 GB/hour to the ~81 GB/hour
case the backend design exists to avoid.

Implementation rules, in order of preference:

1. **Poster frames.** Periodic JPEG snapshots from each publisher, pushed over
   the LiveKit data channel or via the API. Cheap, no subscription, refresh
   every 5–10s.
2. **Subscribe on open, one at a time.** If a live preview is wanted, only while
   the picker is open, only the hovered item, unsubscribed on close.
3. Never a persistent live grid.

Treat any PR that adds a live `<video>` to the rail as a cost regression, not a
feature. This rule belongs in review checklists.

## 3. Switching streams unsubscribes before it subscribes

**Decision: on stage change, unsubscribe the outgoing track, then subscribe the
incoming one.**

The reverse order momentarily doubles every viewer's downstream, and a
reconnect that fails between the two steps leaks a subscription permanently —
which is how a room quietly drifts above its cap with nothing in the UI looking
wrong.

The client is not the enforcement point (the server caps subscriptions via
LiveKit participant permissions), but the client is what makes the enforced cap
feel instant rather than like an error.

## 4. Dark-first

**Decision: dark is the default theme. Light is supported and secondary.**

The functional argument, not a taste argument: the shared screen is the
brightest thing on screen and it is the content. Most shared screens are
documents, IDEs and browsers — large bright rectangles. Dark chrome keeps the UI
from competing with that content and keeps the eye on the stage. Every tool in
this category (OBS, Discord, Twitch, Zoom's dark mode) lands here for the same
reason.

Sessions are now **unlimited length**, which strengthens it: long exposure makes
surrounding luminance a comfort issue rather than a preference.

Both themes ship from the same CSS-variable tokens, so light is not a separate
stylesheet — see [`frontend-stack.md`](frontend-stack.md).

## 5. Sharing is always available, never requested

**Decision: the Share button is enabled for every participant at all times. No
queue, no "request to present", no host approval.**

The backend grants `canPublish` to every admitted participant with no publisher
cap, so any UI implying scarcity — a slot counter, a disabled button, a raise-
hand flow — would be inventing a restriction that does not exist.

What the UI does need: clear indication of **how many people are currently
sharing**, since all ten may be, and the rail is how a viewer navigates them.

## 6. Reserve the stage overlay for the ad break

**Decision: the stage owns a full-cover overlay state from the first build, even
though ads are future work.**

Free users will see an hourly ad break, enforced server-side by revoking
`canSubscribe`. When that happens the stage goes blank whether or not the UI is
ready for it. A stage that can render an explanatory overlay — "ad break,
resuming shortly" — costs little now; retrofitting it means the first ad break
looks like a crash.

The client must treat a server-initiated unsubscribe as a **normal state**, not
an error toast.

## 7. Guest sessions are a first-class entry path

**Decision: public rooms are joinable without login, and the join screen leads
with that.**

The backend issues ephemeral guest sessions for public rooms. Guests cannot own
rooms, cannot access private rooms and are always on the free tier. The UI
should not present login as a wall in front of a public room — the Google
sign-in belongs next to the guest option, not before it.

Private rooms do require sign-in, because an invite check against an anonymous
session is meaningless.

## 8. Quality is shown as achieved, never as promised

**Decision: surface the actual resolution being received, not the plan's label.**

The backend treats 720p/1080p as targets, not guarantees — screen content,
encoder behaviour and network conditions all intervene. A UI badge reading
"1080p" because the user has premium is a support ticket waiting to happen.
Show what the track is actually delivering, and let premium be described as
"up to 1080p".

## Consequences for the repo README

The scope list needs two corrections:

- **"participant grid" → "stage and sharing rail"**, per decision 1.
- Screen sharing controls should name the **picker's poster-frame constraint**,
  since it is a cost boundary rather than an implementation detail.
