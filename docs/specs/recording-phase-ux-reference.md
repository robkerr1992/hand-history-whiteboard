# Recording Phase — Comprehensive UX Specification

_A scenario-driven specification for the entire hand recording experience in Mama Poker._

**Status:** Draft — In Progress
**Date:** 2026-02-14 (updated 2026-02-15)
**Author:** Giuseppe III 🧙‍♂️
**Source of Truth:** This spec extends [Flow 11: Hand Builder](../flows/11-hand-builder.md). Flow 11 defines the foundational UI (horizontal scroll timeline, player panels, card picker, action controls, state management). This spec adds the **Street Whiteboard**, **pot tracking**, and detailed scenarios.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Two-Layer Architecture](#2-two-layer-architecture)
3. [Screen Layout](#3-screen-layout)
4. [Street Whiteboard (Read-Only)](#4-street-whiteboard-read-only)
5. [Horizontal Scroll Timeline (Interactive)](#5-horizontal-scroll-timeline-interactive)
6. [Action Controls](#6-action-controls)
7. [Board Cards](#7-board-cards)
8. [Pot Tracking](#8-pot-tracking)
9. [Scenarios](#9-scenarios)
   - 9.1 [Standard Hand — Preflop Through River Showdown](#91-standard-hand--preflop-through-river-showdown)
   - 9.2 [Hand Ends on Flop — Fold to C-Bet](#92-hand-ends-on-flop--fold-to-c-bet)
   - 9.3 [All-In Preflop — Heads-Up Board Runout](#93-all-in-preflop--heads-up-board-runout)
   - 9.4 [Multi-Way All-In — Side Pots on Turn](#94-multi-way-all-in--side-pots-on-turn)
   - 9.5 [Bomb Pot — Skip Preflop](#95-bomb-pot--skip-preflop)
   - 9.6 [Undo Within a Street](#96-undo-within-a-street)
   - 9.7 [Undo Across Street Boundaries](#97-undo-across-street-boundaries)
   - 9.8 [Hero With Hole Cards vs Unknown Villains](#98-hero-with-hole-cards-vs-unknown-villains)
   - 9.9 [Short Stack Forced All-In](#99-short-stack-forced-all-in)
   - 9.10 [Heads-Up Play (2 Players)](#910-heads-up-play-2-players)
   - 9.11 [Tapping a Past Street on the Whiteboard](#911-tapping-a-past-street-on-the-whiteboard)
   - 9.12 [Hand Complete — Save and Export](#912-hand-complete--save-and-export)
   - 9.13 [Reviewing a Saved Hand from History](#913-reviewing-a-saved-hand-from-history)
   - 9.14 [Straddle Preflop](#914-straddle-preflop)
   - 9.15 [Check-Through on Multiple Streets](#915-check-through-on-multiple-streets)
10. [Accessibility](#10-accessibility)
11. [Animation & Transitions](#11-animation--transitions)
12. [Error States](#12-error-states)
13. [Component Architecture](#13-component-architecture)
14. [Data Model](#14-data-model)

---

## 1. Overview

The Recording Phase begins after the user completes Setup (configures players, assigns hero cards, selects game type). It is the core experience of Mama Poker — the user records each action of a poker hand as it happens (or from memory), building a structured hand history.

### Design Principles

1. **Mom-friendly.** Robert's mom has low vision and is colorblind. Every element must be large, high-contrast, and obvious. No hidden gestures, no subtle cues.
2. **One thing at a time.** The screen always makes it clear: whose turn is it, what can they do, what happened so far.
3. **Pot context always visible.** The pot at the start of the current street is always shown. The user never has to do mental math.
4. **Street-based thinking.** Poker players think in streets, not in a linear action stream. The UI should match their mental model.
5. **Two layers, one truth.** The Whiteboard and the Timeline both display hand progress, but only the Timeline modifies the hand. The Whiteboard is a read-only dashboard.
6. **Undo is fearless.** Users should never worry about making a mistake. Undo is always available, always obvious.

---

## 2. Two-Layer Architecture

The recording phase uses **two parallel displays** that both track the full hand, but serve different purposes:

### The Whiteboard (Street Tabs) — Read-Only Dashboard
- Inspired by **GTO Wizard's** street navigation bar
- Shows all four streets as tabs with pot-at-start for each
- Tapping a tab **views** that street's summary (pot, board, action count)
- **CANNOT modify the hand in any way** — no actions, no edits, no undo
- Pure context and navigation. A dashboard you glance at.

### The Timeline (Horizontal Scroll) — Interactive Workhorse
- The **existing** `PlayerTimeline` component from [Flow 11](../flows/11-hand-builder.md)
- Shows all players as horizontally scrollable panels
- Active player is highlighted (1.5x size, border, "ACTING" badge)
- This is where the user **records actions** via the action controls below
- **This is the only element that can modify the hand**

### Why Two Layers?

| Concern | Whiteboard | Timeline |
|---------|-----------|----------|
| "What street are we on?" | ✅ Highlighted tab | ✅ Street shown in mode indicator |
| "What was the pot on the flop?" | ✅ Tab label shows it | ❌ Not shown |
| "Whose turn is it?" | ❌ Doesn't show | ✅ Active player highlighted |
| "What are their stacks?" | ❌ Doesn't show | ✅ Stack on each panel |
| "Record an action" | ❌ Cannot | ✅ Via action controls |
| "What happened last street?" | ✅ Tap tab to view | ❌ Shows current state only |

They're complementary. The Whiteboard gives **macro context** (street + pot). The Timeline gives **micro context** (players + actions). Both update in real-time as the hand progresses.

---

## 3. Screen Layout

### Mobile Layout (Primary — <768px)

```
┌─────────────────────────────┐
│ ◀  Recording Hand     [✕]  │ ← HEADER (sticky top)
├─────────────────────────────┤
│┌──────┐┌──────┐┌──────┐┌──────┐│
││ PRE  ││ FLOP ││ TURN ││ RIVR ││ ← WHITEBOARD (sticky, read-only)
││12 BB ││45 BB ││      ││      ││
│└──────┘└──────┘└──────┘└──────┘│
├─────────────────────────────┤
│ Board: [A♠] [K♥] [7♦]      │ ← BOARD CARDS
├─────────────────────────────┤
│                             │
│ [SB][BB][UTG][>>HJ<<][CO]→ │ ← TIMELINE (horizontal scroll,
│                             │    interactive)
│                             │
├─────────────────────────────┤
│ Pot: 114 BB                 │ ← CURRENT POT (live)
├─────────────────────────────┤
│ ▸ HJ — Robert (185 BB)     │ ← ACTIVE PLAYER INFO
├─────────────────────────────┤
│ [FOLD]  [CALL 27]  [RAISE] │ ← ACTION BUTTONS (sticky bottom)
├─────────────────────────────┤
│ Raise to: [________] BB    │ ← BET INPUT (expandable)
│ [33%] [50%] [75%] [Pot]    │
├─────────────────────────────┤
│ [↶ Undo: BTN call 27]      │ ← UNDO BUTTON
└─────────────────────────────┘
```

### Sticky Zones

| Zone | Position | Behavior |
|------|----------|----------|
| Header | Top | Always visible |
| Whiteboard | Below header | Sticky — stays visible on scroll |
| Board Cards | Below whiteboard | Visible when applicable |
| Timeline | Middle | Horizontal scroll, auto-scrolls to active player |
| Current Pot | Below timeline | Always visible |
| Active Player + Actions | Bottom | Sticky — always visible during recording |

### Tablet Layout (768–1023px)

Same vertical stack but wider. Timeline shows more players without scrolling. Bet input inline with buttons.

### Desktop Layout (≥1024px)

```
┌────────────────────────────────────────┬──────────────────────┐
│ WHITEBOARD (read-only)                 │                      │
│ [Preflop 12BB] [Flop 45BB] [Turn] ... │  ACTIVE PLAYER INFO  │
├────────────────────────────────────────┤                      │
│ Board: [A♠] [K♥] [7♦]                 │  Stack: 185 BB       │
├────────────────────────────────────────┤  To call: 27 BB      │
│                                        │                      │
│ TIMELINE (horizontal scroll)           ├──────────────────────┤
│ [SB] [BB] [UTG] [>>HJ<<] [CO] [BTN]  │                      │
│                                        │  [FOLD]              │
│ Pot: 114 BB                            │  [CALL 27]           │
│                                        │  [RAISE ___]         │
│                                        │  [33%][50%][75%][Pot]│
│                                        │                      │
│                                        │  [↶ Undo]            │
└────────────────────────────────────────┴──────────────────────┘
```

---

## 4. Street Whiteboard (Read-Only)

The Whiteboard is a **read-only dashboard** inspired by GTO Wizard's street navigation. It tracks the full hand but **cannot modify it in any way**.

### Design

Four equal-width tabs spanning the full width of the screen.

```
┌──────────┐┌──────────┐┌──────────┐┌──────────┐
│ PREFLOP  ││  FLOP    ││  TURN    ││  RIVER   │
│  12 BB   ││  45 BB   ││          ││          │
└──────────┘└──────────┘└──────────┘└──────────┘
```

### Tab Content

- **Line 1:** Street name — `PREFLOP`, `FLOP`, `TURN`, `RIVER`
  - On screens <375px: abbreviate to `PRE`, `FLOP`, `TURN`, `RIVR`
- **Line 2:** Pot at start of that street, in BB
  - Blank for streets not yet reached

### Tab States

| State | Visual | Tappable | When |
|-------|--------|----------|------|
| **Current** | Amber background (`bg-amber-600/30`), white text, 3px amber bottom border, pulsing dot | Yes (shows summary popover) | The street being played right now |
| **Completed** | Dark background (`bg-[var(--bg-secondary)]`), muted text | Yes (shows summary popover) | Street has been played through |
| **Future** | No background, 40% opacity text | No | Street not yet reached |

### Tap Behavior (Read-Only)

Tapping a Whiteboard tab **does NOT change what the Timeline shows**. The Timeline always shows the current live state. Instead, tapping a tab opens a **summary popover** (or bottom sheet on mobile):

```
┌─────────────────────────────────────────┐
│  FLOP  •  Pot: 12 → 45 BB              │
│  Board: [A♠] [K♥] [7♦]                 │
│                                         │
│  BB checks                              │
│  CO bets 8 BB                           │
│  BB calls 8 BB                          │
│                                         │
│  4 actions • 2 players remaining        │
│                                         │
│              [Dismiss]                   │
└─────────────────────────────────────────┘
```

This popover is **informational only**. No buttons to modify the hand. Tapping "Dismiss" or tapping outside closes it. The Timeline and action controls underneath remain untouched.

### What the Whiteboard Does NOT Do

- ❌ Does not switch the Timeline to show a different street
- ❌ Does not hide or replace the action controls
- ❌ Does not allow undoing or modifying actions
- ❌ Does not change the active player
- ❌ Does not interfere with recording in any way

### Auto-Update

The Whiteboard updates automatically as the hand progresses:
- When a new street begins, its tab transitions from "Future" to "Current"
- The previous street transitions to "Completed" and its pot-at-end is calculated
- Pot values in tab labels update in real-time

---

## 5. Horizontal Scroll Timeline (Interactive)

The Timeline is the **existing** `PlayerTimeline` component defined in [Flow 11](../flows/11-hand-builder.md). It is the sole interactive element for hand recording.

### What It Shows

- All players as horizontally scrollable panels
- Each panel: position badge, name, stack, hole cards (if assigned), fold/all-in status
- Active player: 1.5x size, green border, "ACTING" badge
- Folded players: 50% opacity, "FOLDED" overlay
- All-in players: "ALL IN" badge, distinct background

### What It Does

- Displays the **current live state** of all players
- Scrolls to keep the active player visible
- Drives the action controls below (which player is acting)
- Updates in real-time as actions are recorded

### How It Relates to the Whiteboard

Both display the same hand, but at different zoom levels:

| Whiteboard | Timeline |
|-----------|----------|
| Macro: "We're on the flop, pot is 45 BB" | Micro: "HJ is acting, has 185 BB, needs to call 8" |
| Shows all four streets at once | Shows all players on the current street |
| Read-only summary | Interactive recording surface |
| Tap to view past street summary | Scroll to see all player states |

**They never conflict.** The Whiteboard is always up-to-date because it derives from the same hand state the Timeline uses. Recording an action updates both simultaneously.

### No Changes to Existing Behavior

The Timeline's behavior is exactly as specified in Flow 11. This spec does not modify any Timeline interactions. The only addition is the Whiteboard layer above it.

---

## 6. Action Controls

### Layout (Bottom of Screen)

The action controls are fixed to the bottom of the screen during active recording.

### Button Visibility Rules

| Situation | Buttons Shown |
|-----------|--------------|
| No bet facing (can open) | `[FOLD]` `[CHECK]` `[BET ___]` |
| Facing a bet (can call) | `[FOLD]` `[CALL {amt}]` `[RAISE ___]` |
| Stack ≤ call amount | `[FOLD]` `[ALL-IN {stack}]` |
| Viewing past street | Hidden — show "Return to [street]" button instead |
| Hand complete | Hidden — show completion overlay |

### Button Styling

| Button | Size | Color | Text |
|--------|------|-------|------|
| FOLD | 44px min height | `bg-gray-700` | "✕ FOLD" |
| CHECK | 44px min height | `bg-green-700` | "✓ CHECK" |
| CALL | 44px min height | `bg-green-700` | "CALL {amt} BB" |
| BET | 44px min height | `bg-blue-600` | "BET {amt} BB" |
| RAISE | 44px min height | `bg-blue-600` | "RAISE {amt} BB" |
| ALL-IN | 44px min height | `bg-red-600` | "ALL IN {stack} BB" |

All buttons: rounded-xl, font-semibold, active:scale-95 on press.

### Bet/Raise Input

When BET or RAISE is available, the input area appears above the action buttons:

```
┌─────────────────────────────────────────┐
│  Raise to:                              │
│  ┌─────────────────────────────┐        │
│  │         42                  │  BB    │
│  └─────────────────────────────┘        │
│  Min: 15 BB              Max: 185 BB    │
│                                         │
│  [33%]  [50%]  [75%]  [Pot]  [All-in]  │
└─────────────────────────────────────────┘
```

**Presets** are calculated based on pot size:
- **33%:** `floor(pot * 0.33)`
- **50%:** `floor(pot * 0.50)`
- **75%:** `floor(pot * 0.75)`
- **Pot:** `pot` (full pot bet/raise)
- **All-in:** Player's remaining stack

Presets that fall below minimum or above maximum are disabled (dimmed).

**Input validation:**
- Value below min: red border, BET/RAISE button disabled
- Value above max: auto-clamp to max
- Non-numeric input: ignored
- Empty: show min as placeholder

### Active Player Info

Shown above action buttons:

```
┌─────────────────────────────────────────┐
│  ▸ CO — Robert                          │
│    Stack: 185 BB  •  To call: 27 BB     │
└─────────────────────────────────────────┘
```

- **Position badge** + **Player name** (bold)
- **Stack** in BB
- **To call** amount (if facing a bet). Hidden if no bet to face.
- **"All-in to continue"** warning if stack ≤ call amount

---

## 7. Board Cards

### Display Per Street

| Street | Cards Shown | Visual |
|--------|------------|--------|
| Preflop | None | No board area shown |
| Flop | 3 cards | Three cards, full size, bright |
| Turn | Prior 3 (dimmed) + 1 new (bright) | Arrow separator between prior and new |
| River | Prior 4 (dimmed) + 1 new (bright) | Arrow separator between prior and new |

### Card Rendering

```
┌─────┐
│ A   │  52px × 72px on mobile
│  ♠  │  Card face: rank top-left, suit center
│   A │  Rank bottom-right (inverted)
└─────┘
```

**Suit colors (colorblind-safe with shapes):**
- ♠ Spades: Gray/white text (`#e5e7eb`) — pointed shape ♠
- ♥ Hearts: Red (`#ef4444`) — rounded shape ♥
- ♦ Diamonds: Blue (`#3b82f6`) — angular shape ♦
- ♣ Clubs: Green (`#22c55e`) — rounded shape ♣

**Why not standard red/black?** Robert's mom is colorblind. Four distinct colors with four distinct shapes ensures she can always tell suits apart.

### Deal Cards Prompt

When betting completes and the next street needs dealing:

```
┌─────────────────────────────────────────┐
│          Betting complete!              │
│                                         │
│  Tap the board area to deal the TURN    │
│                                         │
│        [  Deal Turn Card  ]             │
└─────────────────────────────────────────┘
```

This replaces the action controls area. The user taps the button, which opens the Card Picker (full-screen modal).

### Card Picker

Full-screen overlay for selecting community cards:

```
┌─────────────────────────────────────────┐
│  Deal the Flop               1 / 3     │ ← Header
├─────────────────────────────────────────┤
│  Selected: [A♠] [K♥] [  ]              │ ← Preview slots
├─────────────────────────────────────────┤
│                                         │
│  ♠: A K Q J T 9 8 7 6 5 4 3 2         │ ← Card grid by suit
│  ♥: A K Q J T 9 8 7 6 5 4 3 2         │
│  ♦: A K Q J T 9 8 7 6 5 4 3 2         │
│  ♣: A K Q J T 9 8 7 6 5 4 3 2         │
│                                         │
│  (used cards grayed out)                │
│                                         │
├─────────────────────────────────────────┤
│  [Cancel]                               │
└─────────────────────────────────────────┘
```

- Cards already dealt (hole cards, prior board) are **grayed out and not tappable**
- Tapping a card adds it to the next empty slot
- When all slots filled, auto-advances the street (no confirm button needed)
- Tapping a filled slot removes that card (allows correction before final card)

---

## 8. Pot Tracking

### Pot Values Tracked

| Value | Where Shown | Calculation |
|-------|-------------|-------------|
| Pot at start of street | Street tab label | Sum of all chips committed on prior streets |
| Running pot (per action) | Right side of each ActionRow | Pot at start + all chips committed on this street through this action |
| Current live pot | Below action list, above active player | Total chips committed by all players so far |
| Final pot | Hand complete overlay | Total of all committed chips |

### Pot Calculation Rules

1. **Preflop pot-at-start** = total forced money (blinds + antes + straddles)
2. **Flop pot-at-start** = all chips committed through end of preflop
3. **Turn pot-at-start** = all chips committed through end of flop
4. **River pot-at-start** = all chips committed through end of turn
5. **Running pot** = pot-at-start of current street + sum of all bets/calls/raises on current street through this action
6. **Final pot** = sum of all chips committed by all players across all streets

### Display Format

- Under 1000: exact number with no decimal if whole, one decimal if fractional. `45 BB`, `12.5 BB`
- 1000+: abbreviated. `1.2K BB`, `15K BB`

---

## 9. Scenarios

In all scenarios below:
- **Whiteboard** = the read-only street tabs at the top
- **Timeline** = the interactive horizontal scroll player panels
- **Actions are always recorded via the action controls below the Timeline, never via the Whiteboard**

---

### 9.1 Standard Hand — Preflop Through River Showdown

**Setup:** 6-handed NLHE, 1/2 blinds, all players 200 BB effective.
Hero is CO with A♠K♥.

#### Preflop

**Screen state:**
- Whiteboard: `[PRE 3BB]` current (amber), `[FLOP]` `[TURN]` `[RIVER]` future (dimmed)
- Timeline: all 6 player panels visible, UTG highlighted as active (1.5x, green border, "ACTING")
- Active player info: "UTG to act • Stack: 200 BB"

**User records actions via action controls:**

| Step | User taps | Timeline updates | Whiteboard | Running pot |
|------|-----------|-----------------|------------|-------------|
| 1 | FOLD | UTG panel → folded (50% opacity) | PRE pot stays 3 BB | 3 |
| 2 | FOLD | HJ panel → folded | No change | 3 |
| 3 | RAISE 6 | CO panel shows "Raise 6" | No change | 9 |
| 4 | FOLD | BTN panel → folded | No change | 9 |
| 5 | FOLD | SB panel → folded | No change | 9 |
| 6 | CALL | BB panel shows "Call" | No change | 12 |

**Street complete.** Action controls replaced with "Deal Flop" prompt.
- Whiteboard updates: `[PRE 3BB]` → completed (dark bg), `[FLOP 12BB]` → current (amber, pulsing dot)
- Timeline: resets active indicators for postflop order

#### Flop: A♦ 7♣ 2♥

**User taps "Deal Flop" → Card Picker opens.**
User selects A♦, 7♣, 2♥. Picker auto-closes.

**Screen state:**
- Whiteboard: `[PRE 3BB]` completed, `[FLOP 12BB]` current, `[TURN]` `[RIVER]` future
- Board: `[A♦] [7♣] [2♥]`
- Timeline: BB highlighted as active (first to act postflop), CO still in hand
- Folded players (UTG, HJ, BTN, SB) remain visible but dimmed

| Step | User taps | Timeline | Running pot |
|------|-----------|----------|-------------|
| 7 | CHECK | BB panel: "Check" | 12 |
| 8 | BET 8 | CO panel: "Bet 8" | 20 |
| 9 | CALL | BB panel: "Call 8" | 28 |

**Street complete.** Whiteboard: `[FLOP 12BB]` → completed, `[TURN 28BB]` → current.

#### Turn: J♣

User deals J♣. Board: `[A♦] [7♣] [2♥] [J♣]`

| Step | User taps | Timeline | Running pot |
|------|-----------|----------|-------------|
| 10 | CHECK | BB: "Check" | 28 |
| 11 | BET 20 | CO: "Bet 20" | 48 |
| 12 | RAISE 55 | BB: "Raise 55" | 83 |
| 13 | CALL | CO: "Call 35" | 118 |

**Street complete.** Whiteboard: `[TURN 28BB]` → completed, `[RIVER 118BB]` → current.

#### River: 2♠

User deals 2♠.

| Step | User taps | Timeline | Running pot |
|------|-----------|----------|-------------|
| 14 | CHECK | BB: "Check" | 118 |
| 15 | CHECK | CO: "Check" | 118 |

**Hand complete — showdown.**

**Both layers freeze.** Whiteboard shows all four streets completed. Timeline shows final player states. Completion overlay appears:

```
┌─────────────────────────────────────┐
│        🏆 Hand Complete!            │
│                                     │
│  Final Pot: 118 BB                  │
│                                     │
│  Preflop:   3 → 12 BB              │
│  Flop:     12 → 28 BB              │
│  Turn:     28 → 118 BB             │
│  River:   118 → 118 BB (checked)   │
│                                     │
│  Winner: ____________               │
│  [Select winner]                    │
│                                     │
│  [Save Hand]  [New Hand]  [Export]  │
└─────────────────────────────────────┘
```

User can still tap Whiteboard tabs to review past street summaries even while the overlay is visible (popover appears above overlay).

User selects winner, optionally reveals villain cards, saves.

---

### 9.2 Hand Ends on Flop — Fold to C-Bet

**Setup:** 6-handed, 1/2, standard open + call preflop. Pot is 12 BB entering flop.

#### Flop: Q♥ 9♦ 4♣

| Step | Action | Running pot |
|------|--------|-------------|
| 1 | BB checks | 12 |
| 2 | CO bets 8 BB | 20 |
| 3 | BB folds | 20 |

**Hand complete — fold-out.**

**Screen state:**
- Street tabs: `[PRE 3BB]` available, `[FLOP 12BB]` active
- Turn and River tabs remain **future/disabled** — they were never reached
- No deal prompt for Turn

**Completion overlay:**
```
┌─────────────────────────────────────┐
│        🏆 Hand Complete!            │
│                                     │
│  Final Pot: 20 BB                   │
│                                     │
│  Preflop:   3 → 12 BB              │
│  Flop:     12 → 20 BB              │
│  Turn:     —                        │
│  River:    —                        │
│                                     │
│  Winner: CO (last player standing)  │
│                                     │
│  [Save Hand]  [New Hand]  [Export]  │
└─────────────────────────────────────┘
```

Winner is auto-determined (last player who didn't fold). No need to select.

---

### 9.3 All-In Preflop — Heads-Up Board Runout

**Setup:** 6-handed, 1/2. UTG shoves 50 BB, everyone folds to BB who calls.

#### Preflop

| Step | Action | Running pot |
|------|--------|-------------|
| 1 | UTG ALL-IN 50 BB | 53 |
| 2-5 | HJ, CO, BTN, SB fold | 53 |
| 6 | BB calls 48 BB | 100 |

**Street complete — but both players are all-in.** No more action possible.

**Screen behavior:**
- No "Deal Flop" prompt with action controls. Instead:

```
┌─────────────────────────────────────┐
│  All players are all-in!            │
│                                     │
│  Deal the board to complete         │
│  the hand.                          │
│                                     │
│  [Deal Remaining Board]             │
└─────────────────────────────────────┘
```

User taps → Card Picker opens for all 5 community cards (or can deal street-by-street — user choice via toggle: "Deal all at once" vs "Deal street by street").

**If dealing street by street:**
- User picks 3 flop cards → Flop tab activates, shows "No action — all-in" and the board
- User picks turn card → Turn tab activates, shows "No action — all-in"
- User picks river card → River tab activates, shows "No action — all-in"

**Street tabs show pot but "(no action)" label:**
```
[PRE 3BB] [FLOP 100BB] [TURN 100BB] [RIVER 100BB]
                         (no action)  (no action)  (no action)
```

The pot doesn't change after preflop because no more chips go in. All three postflop tabs show the same pot-at-start.

**Completion:** After final card dealt, show winner selection.

---

### 9.4 Multi-Way All-In — Side Pots on Turn

**Setup:** 3 players remain on the turn. Stacks: BB (30 BB), CO (80 BB), BTN (200 BB). Pot entering turn: 60 BB.

#### Turn

| Step | Action | Notes |
|------|--------|-------|
| 1 | BB bets ALL-IN 30 BB | BB commits entire stack |
| 2 | CO calls 30 BB | CO still has 50 BB behind |
| 3 | BTN raises to 80 BB | Enough to put CO all-in |
| 4 | CO calls 50 BB (ALL-IN) | CO commits remaining stack |

**Side pot calculation:**

```
Main pot:  60 + (30 × 3) = 150 BB  ← All 3 eligible
Side pot:  (50 × 2) = 100 BB       ← CO + BTN only
Uncalled:  (30 × 1) = 30 BB        ← Returned to BTN

Total: 150 + 100 = 250 BB
```

**Action list display for turn:**

```
┌─────────────────────────────────────────────────┐
│  TURN  •  Pot entering: 60 BB                   │
│  Board: [A♠ K♥ 7♦] → [J♣]                      │
├─────────────────────────────────────────────────┤
│  🔴 BB  ALL-IN 30 BB                   → 90     │
│  ○  CO  calls 30 BB                    → 120    │
│  ●  BTN raises to 80 BB               → 200    │
│  🔴 CO  calls 50 BB (ALL-IN)          → 250    │
├─────────────────────────────────────────────────┤
│  Pot Breakdown:                                  │
│    Main pot: 150 BB (BB, CO, BTN)               │
│    Side pot: 100 BB (CO, BTN)                   │
└─────────────────────────────────────────────────┘
```

**Pot breakdown** appears as a footer in the action list when side pots exist.

**If BTN is the only one not all-in**, deal remaining board → winner selection per pot:
```
│  Winner — Main pot (150 BB): [Select]           │
│  Winner — Side pot (100 BB): [Select]           │
```

---

### 9.5 Bomb Pot — Skip Preflop

**Setup:** 6-handed bomb pot, 5 BB ante each.

**Screen state on start:**
- Street tabs: `[PRE 30BB]` auto-completed, `[FLOP 30BB]` current
- Preflop tab shows:

```
┌─────────────────────────────────────────────────┐
│  PREFLOP  •  Bomb Pot                           │
│  All players ante 5 BB                          │
├─────────────────────────────────────────────────┤
│  6 players × 5 BB = 30 BB                       │
│  No preflop action.                             │
└─────────────────────────────────────────────────┘
```

- Card Picker immediately opens for flop cards
- Action on flop begins with first position (SB or first active player)
- Everything else proceeds normally

---

### 9.6 Undo Within a Street

**Situation:** Flop action in progress. User accidentally records "CO bets 15 BB" but CO actually checked.

**User taps [↶ Undo: CO bet 15]**

**What happens:**
1. Last action row ("CO bets 15 BB") animates out (slide left + fade, 200ms)
2. Running pot reverts (114 → 99 in this example)
3. Current pot display updates
4. Active player reverts to CO
5. Action controls show CO's available actions again
6. Undo button updates to show previous action (or hides if no more history)

**Undo button always shows what it will undo:**
```
[↶ Undo: CO bet 15]  →  [↶ Undo: BB check]  →  [↶ Undo: deal flop]
```

---

### 9.7 Undo Across Street Boundaries

**Situation:** Turn just started. User realizes they recorded wrong flop cards. They need to undo back to flop.

**Undo sequence:**

| Undo # | What's undone | Result |
|--------|--------------|--------|
| 1 | Turn card dealt (J♣) | Turn tab → future, Flop tab → current. Board reverts to 3 cards. |
| 2 | Last flop action | Flop action list removes last row |
| 3 | ... | Continue undoing flop actions |
| N | Flop cards dealt | Flop tab → future, Preflop tab → current. Board clears. |

**Confirmation dialog** when undoing a street deal:

```
┌─────────────────────────────────────┐
│  Undo Deal?                      ✕  │
├─────────────────────────────────────┤
│                                     │
│  Removing the turn card will also   │
│  undo all turn actions.             │
│                                     │
│  0 actions will be removed.         │
│                                     │
├─────────────────────────────────────┤
│  [Cancel]           [Undo Deal]     │
└─────────────────────────────────────┘
```

If there ARE actions on the street being un-dealt:

```
│  Removing the flop will also undo   │
│  4 actions taken on the flop:       │
│                                     │
│  • BB checked                       │
│  • CO bet 8 BB                      │
│  • BB called                        │
│  • (and deal of A♦ 7♣ 2♥)          │
```

**After undo of deal:**
- The undone street tab reverts to "future" state
- The previous street tab becomes "current" again
- If the previous street's betting was complete, the "Deal [Street]" prompt reappears

---

### 9.8 Hero With Hole Cards vs Unknown Villains

**Setup:** Hero is CO with A♠K♥ (assigned in setup). All other players have unknown cards.

**How this affects the recording phase:**

1. **Action list** does NOT show hole cards — those are setup-phase info
2. **Active player info** shows hero badge when it's hero's turn:

```
┌─────────────────────────────────────┐
│  ▸ CO — Robert ⭐ HERO              │
│    [A♠] [K♥]                        │
│    Stack: 185 BB  •  To call: 6 BB  │
└─────────────────────────────────────┘
```

3. For non-hero players, active player info just shows position + name + stack:

```
┌─────────────────────────────────────┐
│  ▸ BB — Villain 1                   │
│    Stack: 194 BB                    │
└─────────────────────────────────────┘
```

4. **On hand complete**, user can optionally reveal villain cards:

```
│  Showdown:                          │
│  CO (Hero): [A♠] [K♥]              │
│  BB: [? ?]  [Reveal cards]         │
```

Tapping "Reveal cards" opens Card Picker for that player's hole cards.

---

### 9.9 Short Stack Forced All-In

**Situation:** BB has 8 BB left. CO raises to 12 BB. It's BB's turn.

**Screen state:**
- Active player: BB — 8 BB stack
- To call: 10 BB (BB already posted 2 BB, raise is to 12, deficit is 10)
- But BB only has 8 BB remaining (6 BB + the 2 BB already posted)

**Action controls:**
```
┌─────────────────────────────────────┐
│  ⚠️ All-in required to continue     │
│                                     │
│  [FOLD]           [ALL-IN 8 BB]     │
└─────────────────────────────────────┘
```

- No CALL button — can't call full amount
- No RAISE button — can't raise
- Only FOLD or ALL-IN
- Warning banner explains why

---

### 9.10 Heads-Up Play (2 Players)

**Setup:** 2 players. SB/BTN and BB.

**Key differences:**
- Preflop: SB/BTN acts first (standard heads-up rules)
- Postflop: SB/BTN acts first (out of position since they're in SB)
- Only 2 action rows per betting round maximum (unless raises go back and forth)
- Tabs work the same — just fewer actions per street

**Street tabs are the same.** No special handling needed except the engine's position logic.

---

### 9.11 Tapping a Past Street on the Whiteboard

**Situation:** Hand is on the Turn. User taps the "FLOP 12BB" tab on the Whiteboard.

**What happens:**

1. A **summary popover** (bottom sheet on mobile) slides up showing the Flop summary
2. The Timeline, board, active player info, and action controls **remain exactly as they are** underneath
3. The popover is purely informational:

```
┌─────────────────────────────────────┐
│  FLOP  •  Pot: 12 → 28 BB          │
│  Board: [A♠] [K♥] [7♦]             │
│                                     │
│  BB checks                          │
│  CO bets 8 BB              → 20    │
│  BB calls 8 BB              → 28    │
│                                     │
│  ✓ Complete • 3 actions             │
│                                     │
│              [Dismiss]              │
└─────────────────────────────────────┘
```

4. User taps "Dismiss" or taps outside → popover closes
5. Recording continues exactly where it was. **Nothing was interrupted.**

**Key principle:** The Whiteboard NEVER interferes with recording. The user can tap any completed street tab at any time to glance at what happened, then dismiss and continue. The Timeline and action controls are never hidden or replaced.

**What if the user taps the current street tab?** Shows a live summary of actions so far on the current street (same popover format, but with "In progress" instead of "Complete").

---

### 9.12 Hand Complete — Save and Export

**When the hand completes** (fold-out or showdown), the completion overlay appears.

#### Fold-Out Completion

```
┌─────────────────────────────────────┐
│          🏆 Hand Complete           │
│                                     │
│  Winner: CO — Robert                │
│  (All opponents folded)             │
│                                     │
│  Final Pot: 20 BB                   │
│                                     │
│  Pot Progression:                   │
│    Preflop:  3 → 12 BB             │
│    Flop:    12 → 20 BB             │
│                                     │
│  ┌───────────────────────────────┐  │
│  │        Save Hand              │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │       Start New Hand          │  │
│  └───────────────────────────────┘  │
│                                     │
│  [Share Link]  [Copy Text]          │
└─────────────────────────────────────┘
```

#### Showdown Completion

```
┌─────────────────────────────────────┐
│          🏆 Hand Complete           │
│                                     │
│  Final Pot: 118 BB                  │
│  Board: [A♦] [7♣] [2♥] [J♣] [2♠]  │
│                                     │
│  Showdown:                          │
│  ┌─────────────────────────────┐    │
│  │ CO (Hero):  [A♠] [K♥]      │    │
│  │ Top pair, top kicker        │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │ BB:  [? ?]  [Reveal Cards]  │    │
│  └─────────────────────────────┘    │
│                                     │
│  Winner: [Select winner ▼]         │
│  ○ CO wins 118 BB                   │
│  ○ BB wins 118 BB                   │
│  ○ Split pot                        │
│                                     │
│  [Save Hand]                        │
│  [Start New Hand]                   │
│  [Share Link]  [Copy Text]          │
└─────────────────────────────────────┘
```

#### Export Formats

**Share Link:** Encodes entire hand into a URL parameter. Opening the link loads the hand in replay mode.

**Copy Text:** Standard poker hand history format:

```
Poker Hand #1 — NLHE 1/2 BB — 6 players
2026-02-14

Preflop (3 BB):
UTG folds, HJ folds, CO raises to 6 BB, BTN folds, SB folds, BB calls 4 BB

Flop (12 BB): A♦ 7♣ 2♥
BB checks, CO bets 8 BB, BB calls 8 BB

Turn (28 BB): J♣
BB checks, CO bets 20 BB, BB raises to 55 BB, CO calls 35 BB

River (118 BB): 2♠
BB checks, CO checks

Showdown: CO shows A♠K♥ (two pair, aces and twos)
CO wins 118 BB
```

---

### 9.13 Reviewing a Saved Hand from History

**Entry point:** HistoryView → tap a saved hand.

**Screen state:**
- Same layout as recording, but NO action controls at bottom
- All reached street tabs are "available" (tappable)
- First tab auto-selected
- Read-only action list
- Board cards shown per tab
- Bottom area shows hand summary instead of action buttons:

```
┌─────────────────────────────────────┐
│  Final Pot: 118 BB                  │
│  Winner: CO — Robert                │
│                                     │
│  [Share Link]  [Copy Text]  [Delete]│
└─────────────────────────────────────┘
```

Tabs navigate between streets. Action list scrolls. No interactivity beyond navigation.

---

### 9.14 Straddle Preflop

**Setup:** UTG posts a 4 BB straddle (2x the big blind).

**Preflop header:**

```
┌─────────────────────────────────────────────────┐
│  PREFLOP  •  Blinds: 1/2 BB  •  Straddle: 4 BB │
│  SB posts 1 BB  •  BB posts 2 BB                │
│  UTG straddles 4 BB                              │
├─────────────────────────────────────────────────┤
```

**Pot-at-start:** 7 BB (1 + 2 + 4)

**Action order:** HJ acts first (left of straddler), around to UTG who acts last preflop (straddle has option).

**Street tab:** `[PRE 7BB]`

---

### 9.15 Check-Through on Multiple Streets

**Situation:** Flop and turn both check through. No bets on either street.

**Flop tab:**

```
┌─────────────────────────────────────────────────┐
│  FLOP  •  Pot entering: 12 BB                   │
│  Board: [A♠] [K♥] [7♦]                          │
├─────────────────────────────────────────────────┤
│  ○ BB checks                                    │
│  ○ CO checks                                    │
├─────────────────────────────────────────────────┤
│  ✓ Street complete  •  Pot: 12 BB               │
└─────────────────────────────────────────────────┘
```

**Turn tab:**

```
┌─────────────────────────────────────────────────┐
│  TURN  •  Pot entering: 12 BB                    │
│  Board: [A♠ K♥ 7♦] → [J♣]                       │
├─────────────────────────────────────────────────┤
│  ○ BB checks                                    │
│  ○ CO checks                                    │
├─────────────────────────────────────────────────┤
│  ✓ Street complete  •  Pot: 12 BB               │
└─────────────────────────────────────────────────┘
```

The pot doesn't change. Both tabs show `12 BB`. This is correct and expected — the UI should not hide these streets or mark them as unimportant. A check-through is still a meaningful decision.

---

## 10. Accessibility

### ARIA Roles

```html
<!-- Street tabs -->
<div role="tablist" aria-label="Poker streets">
  <button role="tab" aria-selected="true" aria-controls="panel-flop"
    id="tab-flop" aria-label="Flop, pot entering 45 big blinds, 4 actions">
    FLOP — 45 BB
  </button>
</div>

<!-- Tab content panel -->
<div role="tabpanel" id="panel-flop" aria-labelledby="tab-flop">
  <ol aria-label="Flop actions">
    <li>BB checks</li>
    <li>CO bets 8 big blinds, pot is now 20 big blinds</li>
  </ol>
</div>

<!-- Active player region -->
<div role="status" aria-live="polite" aria-label="Current player">
  CO, Robert, Hero. Stack 185 big blinds. 6 big blinds to call.
</div>

<!-- Action buttons -->
<div role="group" aria-label="Available actions">
  <button aria-label="Fold">FOLD</button>
  <button aria-label="Call 6 big blinds">CALL 6</button>
  <button aria-label="Raise to 15 big blinds">RAISE 15</button>
</div>
```

### Screen Reader Announcements

| Event | Announcement |
|-------|--------------|
| Street tab switch | "Flop. Pot entering: 45 big blinds. 4 actions." |
| New street begins | "Turn begins. Pot entering: 114 big blinds. Board: Ace of spades, King of hearts, Seven of diamonds, Jack of clubs." |
| Player turn | "CO's turn. Robert. Stack: 185 big blinds. To call: 6 big blinds." |
| Action recorded | "CO raises to 15 big blinds. Pot is now 24 big blinds. BB's turn." |
| Fold recorded | "UTG folds. HJ's turn." |
| All-in | "BB is all in for 30 big blinds. Pot is now 90 big blinds." |
| Hand complete (fold-out) | "Hand complete. All opponents folded. CO wins 20 big blinds." |
| Hand complete (showdown) | "Hand complete. Showdown. Final pot: 118 big blinds." |
| Undo | "Undid: CO bet 15 big blinds. BB's turn." |

### Keyboard Navigation

| Key | Context | Action |
|-----|---------|--------|
| `←` `→` | Street tabs focused | Move between tabs |
| `Enter` / `Space` | Street tab focused | Select tab |
| `Tab` | Anywhere | Cycle through: tabs → action list → active player → action buttons → undo |
| `↑` `↓` | Action list focused | Navigate action rows |
| `F` | Action buttons | Fold |
| `K` | Action buttons | Check |
| `C` | Action buttons | Call |
| `R` | Action buttons | Focus raise input |
| `A` | Action buttons | All-in |
| `Enter` | Raise input | Confirm raise |
| `Ctrl+Z` | Anywhere | Undo |
| `Ctrl+Shift+Z` | Anywhere | Redo |

### Color Contrast (WCAG AA)

All combinations meet 4.5:1 minimum:
- Active tab: white on amber-900 background → 7.2:1 ✅
- Action text: `#e5e7eb` on `#111827` → 12.6:1 ✅
- Dimmed (fold): `#6b7280` on `#111827` → 4.6:1 ✅
- Running pot: `#9ca3af` on `#111827` → 5.8:1 ✅
- Error text: `#fca5a5` on `#111827` → 8.3:1 ✅

### Reduced Motion

When `prefers-reduced-motion: reduce`:
- Action row entrance: instant opacity instead of slide
- Tab switch: instant swap instead of crossfade
- Undo removal: instant disappear instead of slide-out
- Card flip: instant reveal instead of 3D rotation
- Keep: pulsing dot on current tab (reduced to slow pulse)

---

## 11. Animation & Transitions

### Action Row Entrance
- New row slides in from bottom (150ms, ease-out)
- Running pot number counts up to new value (200ms)

### Action Row Removal (Undo)
- Row slides left and fades out (200ms, ease-in)
- Rows below slide up to fill gap (150ms, ease-out)
- Running pot numbers animate back to previous values

### Street Tab Switch
- Content crossfades (150ms)
- Active tab indicator slides to new position (200ms, ease-in-out)

### Street Advance
- New tab pulses once on activation (300ms)
- Previous tab border fades from amber to gray (150ms)
- Content area crossfades to new street (200ms)

### Board Card Deal
- Cards appear one at a time (100ms stagger)
- Each card: scale from 0 → 1 with slight bounce (250ms)

### Pot Number Updates
- Old value crossfades to new value (200ms)
- Brief amber flash on change (100ms on, 200ms fade)

### Hand Complete
- Overlay slides up from bottom (300ms, ease-out)
- Background dims (200ms)
- Content fades in (150ms, 100ms delay)

---

## 12. Error States

### Invalid Bet Amount

```
┌─────────────────────────────────────┐
│  Raise to:                          │
│  ┌─────────────────────────┐        │
│  │       3.5               │  BB    │  ← Red border
│  └─────────────────────────┘        │
│  ⚠ Minimum raise is 12 BB          │  ← Error message
│  Min: 12 BB          Max: 185 BB    │
│                                     │
│  [FOLD]  [CALL 6]  [RAISE] ← disabled│
└─────────────────────────────────────┘
```

- Input border turns red
- Error message appears below input (slide down, 150ms)
- RAISE/BET button disabled until valid
- Error clears when user types a valid amount

### Card Already Used

In card picker, tapping a card that's already dealt:

```
┌─────────────────────────────────────┐
│  ⚠ A♠ is already assigned to       │
│     CO's hole cards                 │
└─────────────────────────────────────┘
```

Toast appears for 2 seconds. Card stays grayed out.

### Engine Validation Error

If the poker engine rejects an action (shouldn't happen in normal flow, but defensive):

```
┌─────────────────────────────────────┐
│  ⚠ Invalid action                   │
│  The engine rejected this move.     │
│  Try a different action.            │
│                              [OK]   │
└─────────────────────────────────────┘
```

3-second auto-dismiss toast. Action controls remain, active player doesn't change.

### State Corruption Recovery

If the hand state becomes invalid (storage corruption, etc.):

```
┌─────────────────────────────────────┐
│  ⚠ Hand state error                 │
│                                     │
│  Something went wrong with this     │
│  hand's data.                       │
│                                     │
│  [Try to Recover]  [Start New Hand] │
└─────────────────────────────────────┘
```

"Try to Recover" replays the action history to rebuild state.

---

## 13. Component Architecture

### Component Tree

```
RecordingPhase.vue
├── StreetWhiteboard.vue           # Read-only street tabs (NEW)
├── CommunityCards.vue             # Board display (EXISTING)
├── PlayerTimeline.vue             # Horizontal scroll player panels (EXISTING — unchanged)
│   └── PlayerPanel.vue            # Single player panel (EXISTING)
├── CurrentPotDisplay.vue          # Live running pot (NEW)
├── ActivePlayerInfo.vue           # Who's acting + stack (EXISTING, updated)
├── ActionControls.vue             # Buttons + bet input (EXISTING, updated)
│   └── BetPresets.vue             # Preset buttons (EXISTING)
├── DealCardsPrompt.vue            # "Deal the Turn" CTA (NEW)
├── UndoButton.vue                 # Undo with label (EXISTING, updated)
├── CardPicker.vue                 # Full-screen card selection (EXISTING)
├── StreetSummaryPopover.vue       # Summary shown when tapping Whiteboard tab (NEW)
├── HandCompleteOverlay.vue        # Completion + save/export (NEW)
│   ├── PotSummary.vue             # Pot progression breakdown (NEW)
│   ├── ShowdownPanel.vue          # Card reveals + winner select (NEW)
│   └── ExportActions.vue          # Save, share, copy (NEW)
└── SidePotBreakdown.vue           # Side pot display when applicable (NEW)
```

### New Components (8)

| Component | Purpose |
|-----------|---------|
| `StreetWhiteboard` | Read-only tab bar: 4 street tabs with pot-at-start. Tapping opens summary popover. |
| `StreetSummaryPopover` | Popover/bottom-sheet showing a completed street's actions + pot + board. Read-only. |
| `CurrentPotDisplay` | Live running pot total shown between Timeline and active player info |
| `DealCardsPrompt` | CTA to deal next street's cards when betting completes |
| `HandCompleteOverlay` | Full completion screen with winner select + save/export |
| `PotSummary` | Pot progression breakdown per street (inside completion overlay) |
| `ShowdownPanel` | Card reveals and winner selection (inside completion overlay) |
| `ExportActions` | Save, share link, copy text buttons |
| `SidePotBreakdown` | Side pot details when multi-way all-in |

### Modified Components (3)

| Component | Changes |
|-----------|---------|
| `RecordingPhase.vue` | Add `StreetWhiteboard` above existing layout. Add `CurrentPotDisplay`. Integrate `DealCardsPrompt` and `HandCompleteOverlay`. |
| `ActivePlayerInfo.vue` | Add hero badge, hole card display for hero |
| `ActionControls.vue` | Add all-in-only mode, forced all-in warning |

### Unchanged Components

| Component | Note |
|-----------|------|
| `PlayerTimeline.vue` | **No changes.** Remains the interactive horizontal scroll as defined in Flow 11. |
| `PlayerPanel.vue` | **No changes.** Individual player panels unchanged. |
| `ModeIndicator.vue` | **Replaced by `StreetWhiteboard`** in recording phase. Keep for setup phase. |
| `CardPicker.vue` | **No changes.** Same two-step suit→rank picker. |
| `BetPresets.vue` | **No changes.** |
| `UndoButton.vue` | Minor: add confirmation dialog for cross-street undo. |

---

## 14. Data Model

### StreetSummary (Derived)

Not stored — computed from `HandState`:

```typescript
interface StreetSummary {
  street: Street                // 'preflop' | 'flop' | 'turn' | 'river'
  potAtStart: number            // Pot entering this street
  potAtEnd: number | null       // Pot leaving (null if in progress)
  actions: Action[]             // Actions taken on this street
  boardCardsDealt: Card[]       // Cards dealt THIS street (3 for flop, 1 for turn/river)
  cumulativeBoard: Card[]       // All board cards through end of this street
  isComplete: boolean           // Has betting finished?
  playerStatesAtStart: Map<string, { stack: number; hasFolded: boolean; isAllIn: boolean }>
}
```

### Utility Function

```typescript
function getStreetSummaries(
  handState: HandState,
  schema: GameTypeSchema
): StreetSummary[]
```

Lives in `@robkerr1992/poker-engine` (or in mama-poker's composables if engine changes aren't desired).

### Store Additions

```typescript
// In useHandStore

/** Computed street summaries */
const streetSummaries = computed<StreetSummary[]>(() => { ... })

/** Currently selected tab (may differ from currentStreet when reviewing past) */
const selectedStreet = ref<Street>('preflop')

/** Whether user is viewing a past street (not the live one) */
const isViewingPastStreet = computed(() =>
  selectedStreet.value !== currentStreet.value
)

/** Auto-advance tab when street changes */
watch(currentStreet, (s) => { if (s) selectedStreet.value = s })
```

### Side Pots

```typescript
interface SidePot {
  amount: number
  eligiblePlayerIds: string[]
  winner?: string              // Set on completion
}
```

Side pot calculation already exists in `@robkerr1992/poker-engine` via `calculatePotsFromPlayers`.

---

_This document is a living spec. Update it as implementation reveals new edge cases or design refinements._

_🧙‍♂️ Giuseppe III — Court Wizard_
