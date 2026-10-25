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

**Decision: rooms are joinable without login — public rooms outright, private
rooms with a code and password. The join screen leads with that.**

The backend issues ephemeral guest sessions. Guests cannot create or own rooms
and are always on the free tier, but a correct join code and room password admit
a guest to a private room without any sign-in. So login is never a wall in front
of a room: the Google button sits beside the guest option, not before it.

Sign-in is only required for creating a room, owning or moderating one, and
entering a private room by **invite** rather than by code.

## 8. The private join screen reveals nothing

**Decision: `/j/{code}` shows a password field and no information about the
room. Not the title, not the description, not the participant count.**

The backend verifies code and password **together** and returns one generic
error for every failure, so that a wrong code cannot be distinguished from a
wrong password. A UI that fetched the title to display a friendly "Entrar em
'Reunião Financeiro'" heading would hand back exactly the enumeration oracle the
backend design refuses to give.

So the private join screen is deliberately plain:

- One form: password only. The code arrives in the link.
- One error message for every failure, matching the server's.
- Title and description appear **after** access is granted, in the room chrome.
- No "checking code…" step, no green tick on the code field.

The frontend cost of this is a slightly colder first impression. That is the
right trade: the alternative leaks which private rooms exist to anyone who wants
to find out.

For **public** rooms the opposite applies — show the title, description and
participant count freely on the join screen, since there is nothing to protect.

## 9. Room identity lives in the chrome

**Decision: the title is always visible in the room header. The description is
one click away, never permanently occupying the layout.**

A title is navigational — with unlimited session length and several rooms open
across tabs, it is how someone knows where they are. It belongs in the header
next to the visibility badge.

A description is context read once on arrival and then noise. Put it behind an
info affordance on the title, or in a collapsible room-details panel. Do not
give it a permanent band across the top: it would push the stage down for the
entire session to show text nobody is reading after minute one, and the stage is
the product.

Both are user-written text rendered to other people: **render as text, never as
HTML**, and never with `dangerouslySetInnerHTML`. Long titles truncate with
ellipsis and carry a `title` attribute rather than wrapping the header onto a
second line.

## 10. Quality is shown as achieved, never as promised

**Decision: surface the actual resolution being received, not the plan's label.**

The backend treats 720p/1080p as targets, not guarantees — screen content,
encoder behaviour and network conditions all intervene. A UI badge reading
"1080p" because the user has premium is a support ticket waiting to happen.
Show what the track is actually delivering, and let premium be described as
"up to 1080p".

## 11. The home page is two different pages

**Decision: `/` renders a landing page for visitors and a room dashboard for
signed-in users. Same route, different page — not a landing page with a
logged-in banner.**

A visitor needs to understand what the product is. A signed-in user needs to get
into a room in one click. Trying to serve both from one layout produces a
marketing hero that a daily user has to scroll past forever.

**The landing page must offer the guest path as a peer to sign-in.** Public rooms
open from a link and private rooms open with a code and password, neither of
which needs an account — so a visitor arriving with a code must never be asked
to sign in first. The landing page carries a join-by-code field next to the
Google button, not behind it.

This follows from decision 7, and it is the easiest decision to lose: every
landing-page template puts a single primary CTA in the hero, and for this product
that CTA would exclude the majority of first-time arrivals.

## 12. Owned and invited rooms are separate sections

**Decision: the dashboard lists "Suas salas" and "Convites" as two sections, not
one merged list.**

The available actions differ, and that is what decides it. On a room you own you
can edit the title, rotate the code, change the password and remove people. On a
room you were invited to you can enter, and nothing else. Merging them produces
cards whose affordances vary row to row, which reads as a bug.

Invited rooms additionally need to show **who invited you** — a room title alone
is not enough context to decide whether to join something.

Sort both by **live activity first**. A room with people in it right now is the
only thing on this page worth acting on immediately; a room that was last used in
March is an archive entry.

## 13. Codes and passwords are masked by default

**Decision: room codes and passwords render masked on the dashboard, revealed
only by an explicit per-room action.**

This is specific to this product and worth stating plainly: **ShareFrame's users
share their screens.** The dashboard is, statistically, one of the most likely
pages in the app to be on someone's screen while ten people watch — either
because they are about to open a room, or because they shared the wrong window.

A dashboard that prints `K7M2Q-9XBVT` and a room password in plain text hands
every private room's credentials to anyone watching, and the owner will not
notice. So:

- Codes and passwords show as `•••••-•••••` with a reveal button per room.
- Revealing is per-room and re-masks on navigation — never a global "show all".
- Copy-to-clipboard works **without** revealing, which is what people actually
  want when sharing a code.
- The same applies to the room settings panel and to any invite dialog.

Treat a plain-text secret in a list view as a defect, not a convenience.

## Consequences for the repo README

The scope list needs two corrections:

- **"participant grid" → "stage and sharing rail"**, per decision 1.
- Screen sharing controls should name the **picker's poster-frame constraint**,
  since it is a cost boundary rather than an implementation detail.
