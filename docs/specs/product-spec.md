# Hand History Whiteboard — Product Specification

**Version:** 1.0 Draft  
**Date:** 2026-02-14  
**Author:** Pythia (for Robert Kerr)

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Data Model](#2-data-model)
3. [Hand Recorder — State Machine & Engine Integration](#3-hand-recorder--state-machine--engine-integration)
4. [Hand Recorder — Interface Design](#4-hand-recorder--interface-design)
5. [Whiteboard View Spec](#5-whiteboard-view-spec)
6. [Annotation System](#6-annotation-system)
7. [Sharing & Groups](#7-sharing--groups)
8. [Technical Architecture](#8-technical-architecture)
9. [Future Considerations](#9-future-considerations)

---

## 1. Product Overview

### What It Is

The **Hand History Whiteboard** is a mobile-first poker hand recorder and visual replay tool. Users record poker hands action-by-action, then review them on a structured "whiteboard" layout that shows community cards, per-street action columns, running stacks, and pot totals.

### Who It's For

| Persona | Use Case |
|---------|----------|
| **Recreational player** | Record interesting hands at the table, review later |
| **Study group member** | Share hands with friends, discuss lines |
| **Coached student** | Post hands for coach review with annotations |
| **Coach** | Annotate student hands at specific decision points |
| **Content creator** | Visual hand breakdowns for social/educational content |

### Core Use Cases

1. **Record** — Capture a complete poker hand (setup → actions → result)
2. **Replay** — View the hand on a structured whiteboard with running stacks and pot
3. **Share** — Send hands to groups or individuals via link
4. **Annotate** — Coach or peer comments on specific action nodes
5. **Collect** — Build a personal hand library, organized by session/group

---

## 2. Data Model

### 2.1 Hand Record Schema

The app extends `HandState` from `@robkerr/poker-engine` with persistence and metadata fields.

```typescript
import type { HandState, GameType, Card, Action, HandResult } from '@robkerr/poker-engine'
import type { Street, Position } from '@robkerr/poker-engine'

/**
 * Persisted hand record — extends engine's HandState with
 * ownership, metadata, and display concerns.
 */
interface HandRecord extends HandState {
  // --- HandState fields (from engine) ---
  // id: string
  // gameType: GameType            // 'nlhe' | 'plo'
  // players: Player[]
  // blinds: { small: number; big: number; ante?: number }
  // heroCards: Card[]
  // board: Card[]
  // currentStreet: Street
  // actions: Action[]
  // result?: HandResult

  // --- Persistence & Metadata ---
  userId: string                   // Owner who recorded the hand
  createdAt: string                // ISO 8601
  updatedAt: string                // ISO 8601
  title?: string                   // Optional user label ("KK vs set on A-high board")
  notes?: string                   // Free-text session note
  tags?: string[]                  // User tags: "bluff", "3bet-pot", "cooler"

  // --- Display Hints ---
  heroPosition?: Position          // Redundant but useful for queries
  heroCardsHidden: boolean         // Whether hero cards default to hidden

  // --- Sharing ---
  visibility: 'private' | 'group' | 'public'
  groupIds?: string[]              // Groups this hand is shared with
  shareToken?: string              // For public link sharing

  // --- Recording State ---
  status: 'recording' | 'complete' // Allows saving incomplete hands
}
```

### 2.2 Annotation Schema

Annotations are keyed to a specific decision point within a hand.

```typescript
interface Annotation {
  id: string
  handId: string

  // Target: which decision node this annotates
  target: AnnotationTarget

  // Content
  authorId: string
  authorRole: 'owner' | 'coach' | 'member'
  body: string                     // Markdown-supported text
  createdAt: string
  updatedAt?: string

  // Thread
  parentId?: string                // For replies within a thread
}

type AnnotationTarget =
  | { type: 'action'; actionIndex: number }     // Specific action (e.g., "Hero raises to 12 on flop")
  | { type: 'street'; street: Street }           // Street-level note ("Flop texture is very dry")
  | { type: 'hand' }                             // General hand-level comment
```

### 2.3 Group Schema

```typescript
interface Group {
  id: string
  name: string
  description?: string
  ownerId: string
  createdAt: string
  avatar?: string                  // URL

  // Settings
  defaultPermission: 'member' | 'viewer'
}

interface GroupMembership {
  groupId: string
  userId: string
  role: 'owner' | 'coach' | 'member' | 'viewer'
  joinedAt: string
}
```

### 2.4 User Schema

```typescript
interface User {
  id: string
  displayName: string
  avatar?: string
  createdAt: string
}
```

### Entity Relationships

```
User ──1:N──> HandRecord
User ──1:N──> Annotation
User ──N:M──> Group (via GroupMembership)
HandRecord ──1:N──> Annotation
HandRecord ──N:M──> Group (via groupIds)
```

---

## 3. Hand Recorder — State Machine & Engine Integration

### 3.1 State Diagram

```
┌─────────┐   tap record    ┌───────────┐  first action   ┌───────────┐
│  IDLE   │────────────────>│ RECORDING │────────────────>│ RECORDING │
└─────────┘  (auto-setup:   │ (preflop) │  (stacks lock   │ (active)  │
              9p NLHE,       └─────┬─────┘   per-player)   └─────┬─────┘
              default stacks)      │                             │
                                   │                             │
                          street complete?                  action recorded
                                   │                             │
                              ┌────▼─────┐                 ┌─────▼─────┐
                              │  BOARD   │  board card      │ AWAITING  │
                              │  INPUT   │  panel in        │  ACTION   │
                              │  PANEL   │  timeline        └───────────┘
                              └────┬─────┘
                                   │
                              board entered
                                   │
                            ┌──────▼──────┐
                            │  NEXT       │──> back to AWAITING ACTION
                            │  STREET     │    (or COMPLETE if river done)
                            └──────┬──────┘
                                   │
                       ┌───────────▼───────────┐
                       │      COMPLETE         │
                       │  (fold / showdown /   │
                       │   all-in runout)      │
                       └───────────────────────┘
```

### 3.2 Setup — Inline Panel (No Dedicated Setup Screen)

There is **no dedicated setup screen**. The first panel in the horizontal timeline IS the setup panel.

- **Defaults:** 9-player NLHE, standard blinds, all stacks at default (100bb)
- **Display:** Shows a game config summary (game type, player count, blinds, stacks) — same shape as GTO Wizard's left info panel
- **"Edit" button** opens a modal for changing: game type, player count, blinds, stack sizes, ante
- **Zero friction:** User taps "Record" → lands on timeline with setup panel + first action panel already visible. No blocking setup step.

**Player names** default to position labels (UTG, HJ, CO, BTN, SB, BB, etc.). User can optionally rename any player.

**Configurable fields (via Edit modal):**

| Field | Default | Notes |
|-------|---------|-------|
| Game type | `'nlhe'` | PLO future |
| Player count | 9 | 2–10 |
| Player names | Position labels | Optional rename |
| Positions | Auto | `getPositionsForCount(n)` from engine |
| Stack sizes | 100bb each | Per player, individually adjustable |
| Blinds (SB/BB) | 1/2 | |
| Ante | 0 | |
| Hero seat | None | Which player is hero |

**Engine integration:**

```typescript
import {
  getPositionsForCount,
  createValidationEngine,
  nlheSchema,
} from '@robkerr/poker-engine'

// 1. Get positions for table size
const positions = getPositionsForCount(9)

// 2. Build initial HandState with defaults
const handState: HandState = {
  id: generateId(),
  gameType: 'nlhe',
  players: positions.map((pos, i) => ({
    id: `player-${i}`,
    name: playerNames[i] || pos,  // Default name is position label
    position: pos,
    stackSize: stacks[i] ?? 200,  // Default 100bb = 200 chips at 1/2
    isHero: i === heroIndex,
  })),
  blinds: { small: 1, big: 2, ante: 0 },
  heroCards: [],
  board: [],
  currentStreet: 'preflop',
  actions: [],
}

// 3. Create validation engine
const engine = createValidationEngine('nlhe')
```

### 3.3 Action Recording Loop

Each iteration:

1. **Query who acts next** and what they can do
2. **User selects action** + enters amount (if bet/raise)
3. **Validate** the action
4. **Apply** to HandState
5. **Check** if betting round is complete

```typescript
// 1. Get decision state
const decisionState = engine.getDecisionState(handState)
if (!decisionState) {
  // Betting round complete — transition street or complete hand
}

const { activePlayer } = decisionState

// 2. Get available actions
const available = engine.getAvailableActions(handState)
// => [
//   { type: 'fold' },
//   { type: 'call', callAmount: 2 },
//   { type: 'raise', minAmount: 4, maxAmount: 200 },
// ]

// 3. Get bet constraints (for bet/raise amount input)
const constraints = engine.getBetConstraints(handState)
// => { min: 4, max: 200, callAmount: 2, isAllIn: false }

// 4. User picks action
const action: Action = {
  type: 'raise',
  playerId: activePlayer.id,
  amount: 6,              // Free-form input from user
  street: handState.currentStreet,
}

// 5. Validate
const result = engine.validateAction(action, handState)
if (!result.valid) {
  showError(result.error)  // e.g., "Raise must be at least 4"
  return
}

// 6. Apply — append to actions array (immutable)
handState = {
  ...handState,
  actions: [...handState.actions, action],
}

// 7. Check if round complete
if (engine.isBettingRoundComplete(handState)) {
  transitionToNextStreet()
}
```

### 3.4 Street Transitions

When `isBettingRoundComplete()` returns true:

```typescript
import { getNextStreet, advanceStreet } from '@robkerr/poker-engine'

// Check if hand is over (fold-out)
const nonFolded = playerStates.filter(p => !p.hasFolded)
if (nonFolded.length <= 1) {
  completeHand('fold')
  return
}

// Check if all remaining players are all-in (runout)
const canAct = nonFolded.filter(p => !p.isAllIn)
if (canAct.length <= 1) {
  // All-in runout — user enters remaining board cards
  completeAsRunout()
  return
}

// Normal street transition
const nextStreet = getNextStreet(handState.currentStreet)
if (!nextStreet) {
  // River betting complete → showdown
  completeHand('showdown')
  return
}

// Prompt user for board cards
promptBoardCards(nextStreet)
// Flop → 3 cards, Turn → 1 card, River → 1 card
```

**Board card entry:**

```typescript
import { getExpectedCardCount } from '@robkerr/poker-engine'

const cardsNeeded = getExpectedCardCount(nextStreet)
// 'flop' → 3, 'turn' → 1, 'river' → 1

// After user enters cards:
handState = {
  ...handState,
  board: [...handState.board, ...newCards],
  currentStreet: nextStreet,
}
```

### 3.5 Hand Completion & Winner Determination

Three outcomes:

| Outcome | Trigger | Winner Determination |
|---------|---------|---------------------|
| **Fold** | All but one player folds | **Auto** — engine detects last player standing, no cards needed |
| **Showdown (all cards known)** | River betting completes, all remaining players' cards entered | **Auto-evaluate** — hand evaluator determines winner from hole cards + board |
| **Showdown (unknown cards)** | River betting completes, one or more opponents mucked / cards unknown | **Manual selection** — user picks winner, loser's cards marked as unknown |
| **All-in runout** | All players all-in before river | Remaining board cards entered, then auto-evaluate or manual as above |

#### Winner Determination Logic

**Auto-evaluate is the default.** When all remaining players' hole cards and all board cards are known, the engine's hand evaluator (`GameTypeSchema.evaluator`) automatically determines the winner of each pot. Manual override is always available.

```typescript
import { evaluateShowdown } from '@robkerr/poker-engine'

// Fold-out: engine detects last player standing
const nonFolded = playerStates.filter(p => !p.hasFolded)
if (nonFolded.length === 1) {
  const result: HandResult = {
    outcome: 'fold',
    pots: [{ id: 'main', amount: pot, winners: [nonFolded[0].id], winAmount: pot, isSplit: false }],
  }
  handState = { ...handState, result, status: 'complete' }
  return
}

// Showdown: all cards known → auto-evaluate
const allCardsKnown = nonFolded.every(p => shownCards[p.id]?.length > 0)
if (allCardsKnown) {
  const result = evaluateShowdown(handState, shownCards)
  // Auto-populated with hand ranks, best five, and pot winners
  handState = { ...handState, result, status: 'complete' }
  return
}

// Showdown: unknown cards → manual winner selection
// User selects winner; loser's cards stored as unknown
const result: HandResult = {
  outcome: 'showdown',
  pots: [{ id: 'main', amount: 120, winners: [manualWinnerId], winAmount: 120, isSplit: false }],
  shownCards: {
    [manualWinnerId]: [{ rank: 'A', suit: 's' }, { rank: 'K', suit: 's' }],
    [loserId]: 'unknown',  // Opponent mucked
  },
}
handState = { ...handState, result, status: 'complete' }
```

### 3.6 Undo System

Two complementary undo mechanisms:

#### Undo Button
Dedicated undo button in the recorder UI. Undoes the last recorded action and steps the timeline back one panel.

#### Scroll-Back Rewind
User can scroll back through the timeline to any previously-acted panel and change the action or bet size. This **auto-undoes all subsequent actions** from that point forward. The engine replays from scratch using `buildDecisionState` on the truncated action list, so this is always consistent.

**Implementation:**

```typescript
function undoToIndex(handState: HandState, targetIndex: number): HandState {
  const actions = handState.actions.slice(0, targetIndex)

  // Recompute board and street from truncated actions
  let board: Card[] = []
  let currentStreet: Street = 'preflop'

  // Engine replays the truncated action list to rebuild state
  // buildDecisionState handles street transitions, board state, etc.

  return { ...handState, actions, board: trimBoardToStreet(handState.board, currentStreet), currentStreet, result: undefined }
}

function undo(handState: HandState): HandState {
  return undoToIndex(handState, handState.actions.length - 1)
}
```

**Stack unlocking on undo:** If undoing past a player's first action, their stack field becomes editable again (see §4.6 Per-Player Stack Locking).

The engine's `buildDecisionState` will correctly replay the truncated action list to reconstruct `DecisionState`.

---

## 4. Hand Recorder — Interface Design

This section describes the recorder's visual interface and interaction model. The recorder uses a **horizontal scrolling timeline** of action panels — the same mental model as the read-only whiteboard, but interactive.

### 4.1 Timeline Layout

```
┌────────┬────────┬────────┬────────┬────────┬────────┬─ ─ ─ ─┐
│ SETUP  │ SB     │ BB     │ UTG    │ HJ     │ CO     │ BTN    │
│ 9p     │ Post 1 │ Post 2 │ Fold ✓ │ Fold ✓ │ Raise  │ ●ACT  │
│ NLHE   │        │        │        │        │ 6      │        │
│ [Edit] │ [199]  │ [198]  │ [100]  │ [100]  │ [94]   │        │
└────────┴────────┴────────┴────────┴────────┴────────┴─ ─ ─ ─┘
                                                    ──────────>
                                              scrolls left as
                                              actions confirm
```

- Each **decision point** is a panel, scrolling left to right
- The **active panel** (rightmost) is highlighted with a border/glow; its action buttons are interactive
- **Acted panels** show ALL available actions with the chosen one highlighted and others muted (preserves decision tree context for study)
- After an action is confirmed → panel slides left, next player's panel appears on the right
- **Community cards** display above the timeline, filling in as streets deal

### 4.2 Action Panel Design

Each panel follows the same shape as GTO Wizard action panels:

- Shows **ALL available actions** for that decision point (fold, call, raise, etc.)
- **Active panel:** highlighted border, buttons interactive
- **Acted panel:** chosen action highlighted in its color, all other options visible but muted
- This design preserves the full decision tree — when reviewing, you can see what options existed at each point

### 4.3 Bet/Raise Input

When the user taps Bet or Raise on the active panel:

- **Inline input field** opens within the panel (not a modal or separate screen)
- **Pre-filled** with the minimum legal amount (from `engine.getBetConstraints()`)
- **Free-form number input** — user types exact chip/BB amount, no presets
- **Soft validation:** if the entered amount is below min raise or exceeds stack, shows a warning message (e.g., "Minimum raise is 50") and disables the confirm button
- **Confirm button** to lock in the bet amount
- Cannot go below min raise or exceed stack — this teaches the rules

### 4.4 Skip-Forward

For speed when recording hands with many preflop folds:

- User can **scroll right past the active panel** to a future player's panel and select an action (fold/check)
- All **intermediate players auto-fill** with the contextual default action:
  - Fold if facing a bet/raise
  - Check if no bet to face
- Massive speed improvement — recording "folds to CO" is one tap instead of six

### 4.5 Consecutive Fold Compression

When multiple consecutive folds occur (especially common via skip-forward):

```
┌────────┐  ┌──────────────────┐  ┌────────┐
│ BB     │  │ ┌─UTG Fold [100]│  │ CO     │
│ Post 2 │  │ ├─HJ  Fold [100]│  │ Raise  │
│        │  │ ├─CO  Fold [100]│  │ 6      │
│ [198]  │  │ └─BTN Fold [100]│  │ [94]   │
└────────┘  └──────────────────┘  └────────┘
             compressed panel       full panel
             (expandable)
```

- **2+ consecutive folds** compress into a single narrow stacked panel
- Each fold is still a visible row within the stack (position + fold + stack)
- **Tapping any individual row** rewinds to that player's decision: panel expands, all subsequent actions undo
- **Single folds** between real actions stay as full-size panels
- Keeps the hand's story visually clear — real decisions get full panels, dead space compresses

### 4.6 Per-Player Stack Locking

Stacks are **not** locked globally at the start of recording. Instead:

- Each player's stack **locks when THEIR first action is recorded**
- **Before acting:** stack field is editable (tap to change)
- **After first action:** stack shows 🔒, non-editable
- **Undo past that action:** stack unlocks again

**Rationale:** More precise, allows partial knowledge. You may not know UTG's stack if they fold preflop without showing, but you do know BTN's stack when they raise. This avoids forcing the user to fill in data they don't have.

### 4.7 Card Assignment

Hole cards can be assigned to **any player at any point** during recording:

- Tap a player's panel → option to assign hole cards
- **Rank + suit selector:** tap rank → tap suit → card placed (same selector as board cards)
- Only unused (available) cards shown in the selector
- **"Mucks" button** — marks a player's cards as unknown (for opponents who don't show at showdown)
- Hero cards can be hidden/shown via a visibility toggle

### 4.8 Street Transitions — Board Card Input

When a betting round completes, a **board card input panel** appears in the timeline flow (not a modal):

```
┌────────┬───────────────────────┬────────┐
│ BB     │ 🃏 FLOP              │ BB     │
│ Call 6 │ [A♠] [K♥] [7♦]      │ ●ACT  │
│ [194]  │ rank+suit selector   │        │
└────────┴───────────────────────┴────────┘
```

- **Rank + suit selector:** tap rank → tap suit → card placed
- **Flop:** 3 card picks. **Turn:** 1 pick. **River:** 1 pick.
- Already-used cards (hole cards, previous board cards) are disabled in the selector
- Community cards also display **above the timeline** and fill in as streets deal

### 4.9 Undo in the Timeline

Two complementary mechanisms (see §3.6 for implementation):

1. **Undo button** — always visible, undoes the last action, steps timeline back one panel
2. **Scroll-back rewind** — scroll left to any past acted panel, change the action → auto-undoes everything after that point and replays from the new choice

Both work because the engine's `buildDecisionState` replays from scratch — just truncate the action list and rebuild.

---

## 5. Whiteboard View Spec

### 5.1 Layout Structure

```
┌─────────────────────────────────────────────────────────────┐
│                     COMMUNITY CARDS                          │
│   ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐         Pot: $120    │
│   │ A♠ │ │ K♥ │ │ 7♦ │ │ ?  │ │ ?  │                      │
│   └────┘ └────┘ └────┘ └────┘ └────┘                      │
├─────────────────────────────────────────────────────────────┤
│                      ACTION COLUMNS                         │
│                                                             │
│ PREFLOP (6)  │ FLOP (4)     │ TURN (0)    │ RIVER (0)     │
│ ─────────── │ ────────────  │ ──────────  │ ──────────    │
│ SB  📤 Post 1   [199]│ BB  ✅ Check   [88]│             │              │
│ BB  📤 Post 2   [198]│ UTG 🟡 Bet 15  [85]│             │              │
│ UTG 🔴 Fold     [100]│ CO  🟢 Call 15 [73]│             │              │
│ CO  🟢 Call 2    [98]│ BB  🔴 Fold    [88]│             │              │
│ BTN 🔴 Fold     [100]│                    │             │              │
│ ★ [A♠ K♥]            │                    │             │              │
│ SB  🔴 Fold     [199]│                    │             │              │
│ BB  ✅ Check     [198]│                    │             │              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Community Cards Section

- **5 card slots** always rendered
- Dealt cards show rank + suit with suit color (4-color deck)
- Undealt cards show dashed border with `?` placeholder
- **Pot total** in top-right corner — running pot calculated via `calculatePot(playerStates)`
- Cards fill left-to-right as streets deal: slots 0-2 = flop, slot 3 = turn, slot 4 = river

### 5.3 Action Columns

**Layout:** Horizontal scroll container with 4 fixed columns: PREFLOP | FLOP | TURN | RIVER

**Street Header:**
- Street name in caps
- Action count badge: e.g., `PREFLOP (6)`
- Streets with no actions yet show grayed header, no rows

**Per-Action Row Format:**

```
┌──────────┬──────┬──────────────┬─────────┐
│ Position │ Icon │ Action+Amt   │ Stack   │
├──────────┼──────┼──────────────┼─────────┤
│ UTG      │ 🟡   │ Raise 12     │ [188]   │
│ CO       │ 🟢   │ Call 12      │ [88]    │
│ BTN      │ 🔴   │ Fold         │ [100]   │
└──────────┴──────┴──────────────┴─────────┘
```

### 5.4 Color Coding

| Action | Color | Icon |
|--------|-------|------|
| Fold | Red `#dc2626` | 🔴 |
| Check | Green `#16a34a` | ✅ |
| Call | Green `#16a34a` | 🟢 |
| Bet | Yellow/Amber `#d97706` | 🟡 |
| Raise | Yellow/Amber `#d97706` | 🟡 |
| All-in | Purple `#7c3aed` | 🟣 |
| Post blind | Gray `#6b7280` | 📤 |

### 5.5 Running Stacks

Each action row shows the player's stack **after** that action. Calculated by replaying actions up to that index using `applyAction()`.

Implementation: use `replayToPlayerStates()` incrementally, or precompute a snapshot array on hand load:

```typescript
import { initializePlayerStates, applyAction, resetForNewStreet } from '@robkerr/poker-engine'

function computeStackSnapshots(hand: HandRecord): Map<number, Map<string, number>> {
  let states = initializePlayerStates(hand.players, hand.blinds)
  let currentBet = hand.blinds.big
  let lastStreet: Street = 'preflop'
  const snapshots = new Map<number, Map<string, number>>()

  for (let i = 0; i < hand.actions.length; i++) {
    const action = hand.actions[i]

    if (action.street && action.street !== lastStreet) {
      states = resetForNewStreet(states)
      currentBet = 0
      lastStreet = action.street!
    }

    if (action.type === 'bet' || action.type === 'raise') {
      currentBet = action.amount ?? currentBet
    } else if (action.type === 'all-in' && (action.amount ?? 0) > currentBet) {
      currentBet = action.amount!
    }

    states = applyAction(states, action, currentBet)

    const stackMap = new Map<string, number>()
    for (const s of states) {
      stackMap.set(s.id, s.currentStack)
    }
    snapshots.set(i, stackMap)
  }

  return snapshots
}
```

### 5.6 Pot Display

- Show running pot total in the community cards section
- Pot is derived from `calculatePot(playerStates)` — sum of (startingStack − currentStack) for all players
- On hand completion, show final pot and winner info

### 5.7 Showdown Rendering

At the end of the RIVER column (or wherever the hand ends):

```
┌──────────────────────────────┐
│ 🏆 SHOWDOWN                  │
│ CO  [A♠ K♠]  ← Winner $120  │
│ BB  [Q♥ Q♦]  ← Mucked       │
└──────────────────────────────┘
```

For fold-out endings, the last action in the column shows the fold and a "wins $X" label on the remaining player.

### 5.8 Hero Card Visibility Toggle

- Toggle button (eye icon) in the header or near hero's card display
- When hidden: hero cards show as `[? ?]` (same placeholder style as undealt board cards)
- Default: `heroCardsHidden` field on `HandRecord`
- Useful when sharing hands to avoid spoilers or for quiz-style review

### 5.9 Board Card Display

| State | Rendering |
|-------|-----------|
| Dealt | Full card face with rank + colored suit symbol |
| Not yet dealt | Dashed border, `?` centered, muted gray |
| Card being entered | Input mode (card picker / text input) |

---

## 6. Annotation System

### 9.1 Overview

Annotations live on a **separate tab** from the whiteboard. The hand detail view has two tabs:

```
┌──────────────────────────────────┐
│  [📋 Whiteboard]  [💬 Notes]     │
└──────────────────────────────────┘
```

### 9.2 Notes Tab Layout

The notes tab mirrors the whiteboard structure but shows annotation threads instead of stack info:

- **Hand-level notes** at top
- **Street-level notes** as section headers
- **Action-level notes** attached to specific action rows

Each annotatable node shows a count badge on both tabs. Tapping an action on the whiteboard can jump to its annotation thread.

### 9.3 Thread Model

```
Hand-level comment
├── Reply 1
└── Reply 2

Preflop > Action 3 (UTG raises to 6)
├── Coach: "This is a standard open from UTG. 3x is fine at this stack depth."
└── Student: "Should I go larger with KK here?"
    └── Coach: "No, 3x is standard. Larger sizing tells the story."
```

### 9.4 Roles & Permissions

| Role | Can annotate | Can see annotations | Can delete |
|------|-------------|-------------------|-----------|
| Owner | ✅ | All | Own |
| Coach | ✅ | All | Own |
| Member | ✅ | All | Own |
| Viewer | ❌ | All | — |

Coaches are designated at the **group** level. A user is a coach if their `GroupMembership.role === 'coach'` for any group the hand is shared with.

---

## 7. Sharing & Groups

### 9.1 Sharing Methods

| Method | How | Access |
|--------|-----|--------|
| **Private** | Default — only owner sees | Owner only |
| **Direct link** | Generate `shareToken`, anyone with link can view | Viewer access |
| **Group** | Post hand to one or more groups | Per group role |

### 9.2 Group Concept

Groups are lightweight containers for organizing study/coaching communities.

- **Study group** — peers share and discuss hands (all members)
- **Coaching group** — coach reviews student hands (coach + students)

### 9.3 Permissions Model

```
OWNER > COACH > MEMBER > VIEWER

Owner:   Full control (edit, delete, manage sharing)
Coach:   Annotate, view all hands in group
Member:  Share own hands, annotate, view shared hands
Viewer:  Read-only (link access)
```

### 9.4 Group Features

- Invite by link or username
- Hand feed (chronological list of shared hands)
- Member list with roles
- Group settings (name, description, default role for new members)

---

## 8. Technical Architecture

### 9.1 Frontend Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | **Vue 3** + Composition API | Consistent with mama-poker |
| Mobile | **Capacitor** | iOS + Android from single codebase |
| State | **Pinia** | Reactive stores for hand recording state |
| UI | **Tailwind CSS** | Utility-first, mobile-optimized |
| Engine | `@robkerr/poker-engine` | Client-side — all game logic runs in the app |

### 9.2 Engine Integration Pattern

The engine runs **entirely client-side**. The server never validates poker logic — it stores and serves hand records.

```
┌─────────────────────────────────────┐
│            Vue Frontend             │
│                                     │
│  ┌──────────┐    ┌───────────────┐  │
│  │ Recorder │───>│ Poker Engine  │  │
│  │  Store   │<───│ (validation,  │  │
│  │ (Pinia)  │    │  state calc)  │  │
│  └──────────┘    └───────────────┘  │
│       │                             │
│       │ save/load                   │
│       ▼                             │
│  ┌──────────┐                       │
│  │ API Layer│                       │
│  └────┬─────┘                       │
└───────┼─────────────────────────────┘
        │ HTTPS
        ▼
┌──────────────┐
│  Laravel API │
│  (Routes +   │
│  Controllers)│
├──────────────┤
│  Eloquent    │
│  ORM         │
├──────────────┤
│  MySQL       │
└──────────────┘
│  Deployed via Laravel Forge
```

### 9.3 Key Pinia Stores

```typescript
// stores/recorder.ts
interface RecorderState {
  handState: HandState | null
  engine: ValidationEngine | null
  status: 'idle' | 'recording' | 'board-input' | 'complete'
  undoStack: HandState[]       // For multi-level undo
}

// stores/handLibrary.ts
interface HandLibraryState {
  hands: HandRecord[]
  loading: boolean
  filters: { gameType?: GameType; tags?: string[] }
}
```

### 9.4 Laravel API Architecture

The backend is a **Laravel** application with API routes, Eloquent models, and JSON resource transformers. Deployed via **Laravel Forge**.

#### Eloquent Models

| Model | Table | Key Relations |
|-------|-------|---------------|
| `Hand` | `hands` | `belongsTo(User)`, `hasMany(Annotation)`, `belongsToMany(Group)` |
| `Annotation` | `annotations` | `belongsTo(Hand)`, `belongsTo(User)` |
| `Group` | `groups` | `belongsTo(User, 'owner_id')`, `belongsToMany(User, 'group_memberships')` |
| `GroupMembership` | `group_memberships` | Pivot with `role` column |
| `User` | `users` | `hasMany(Hand)`, `belongsToMany(Group)` |

#### API Routes (`routes/api.php`)

All routes are prefixed `/api` and use Laravel Sanctum for auth.

**Hands** — `HandController` + `HandResource`

```
POST   /api/hands                    store()
GET    /api/hands                    index()    — paginated, filterable
GET    /api/hands/{hand}             show()
PUT    /api/hands/{hand}             update()
DELETE /api/hands/{hand}             destroy()

GET    /api/hands/shared/{token}     showByToken()  — public, no auth
POST   /api/hands/{hand}/share       share()
```

**Annotations** — `AnnotationController` + `AnnotationResource`

```
GET    /api/hands/{hand}/annotations              index()
POST   /api/hands/{hand}/annotations              store()
PUT    /api/hands/{hand}/annotations/{annotation} update()
DELETE /api/hands/{hand}/annotations/{annotation} destroy()
```

**Groups** — `GroupController` + `GroupResource`

```
POST   /api/groups                          store()
GET    /api/groups                          index()
GET    /api/groups/{group}                  show()
PUT    /api/groups/{group}                  update()
DELETE /api/groups/{group}                  destroy()

GET    /api/groups/{group}/hands            hands()
POST   /api/groups/{group}/members          addMember()
DELETE /api/groups/{group}/members/{user}   removeMember()
PUT    /api/groups/{group}/members/{user}   updateRole()

POST   /api/groups/{group}/invite           invite()
POST   /api/groups/join/{token}             join()
```

**Users** — `UserController`

```
GET    /api/users/me                 show()
PUT    /api/users/me                 update()
```

### 9.5 Storage Considerations (MySQL)

- **HandRecord** stores the complete action log — a single `JSON` column for the engine-compatible `HandState` fields, plus relational columns for query/filter fields (`user_id`, `game_type`, `created_at`, `visibility`, `status`)
- **Annotations** are relational rows (indexed by `hand_id` + `target`)
- **Board and actions** are arrays stored as MySQL `JSON` columns — no need to normalize individual actions into rows
- Estimated record size: ~2-5 KB per hand (JSON). At 10,000 hands per active user this is ~50 MB — well within MySQL comfort zone
- MySQL 8.0+ `JSON` type supports indexing via generated columns for query-critical paths

---

## 9. Future Considerations

### 9.1 PLO Support

The engine already has `ploSchema` registered. Architecture is ready:

```typescript
// PLO just works — different schema, same ValidationEngine
const engine = createValidationEngine('plo')

// Schema enforces:
// - 4 hole cards (schema.holeCards === 4)
// - Pot-limit betting structure
// - Must use exactly 2 hole cards + 3 board cards (evaluation)
```

UI changes needed:
- Card picker supports 4 hole cards (`HoleCardLayout: 'horizontal-4'`)
- Whiteboard hero card display widens for 4 cards
- Game type selector in setup

### 9.2 Tournament Support

- Add `format: 'cash' | 'tournament'` to HandRecord
- Tournament-specific fields: level, antes, ICM considerations
- Blind schedule reference

### 9.3 Hand Import

Parse common hand history formats:

| Source | Format | Priority |
|--------|--------|----------|
| PokerStars | Text (well-documented) | High |
| GGPoker | Text / JSON | Medium |
| 888poker | Text | Low |
| Manual (other apps) | Standardized JSON | Medium |

Parser maps external format → `HandState` + metadata → `HandRecord`.

### 9.4 Analytics & Stats

Once a hand library exists, derive:

- VPIP/PFR by position
- Aggression frequency by street
- Win rate by hand category
- Common leak patterns (check-fold frequency, etc.)
- Filter by tags, date range, stakes

### 9.5 Hand Evaluation (`@robkerr/poker-engine`)

Hand strength evaluation lives in the engine package, behind the `GameTypeSchema` interface so each game type provides its own evaluator.

#### Evaluator Interface

```typescript
interface HandEvaluator {
  /** Determine best 5-card hand from hole cards + board */
  evaluate(holeCards: Card[], board: Card[]): EvaluatedHand

  /** Compare two evaluated hands. Returns -1, 0, or 1. */
  compare(a: EvaluatedHand, b: EvaluatedHand): number
}

interface EvaluatedHand {
  rank: number            // Numeric rank (1 = Royal Flush … 10 = High Card)
  name: string            // "Royal Flush", "Straight", "Two Pair", etc.
  bestFive: Card[]        // The 5 cards that make the hand
}
```

#### Game Type Implementations

- **NLHE:** Best 5 from any combination of 2 hole + 5 board cards
- **PLO:** Must use exactly 2 of 4 hole cards + exactly 3 board cards (enforced by `ploSchema.evaluator`)

Each `GameTypeSchema` exposes its evaluator:

```typescript
interface GameTypeSchema {
  // ... existing fields ...
  evaluator: HandEvaluator
}
```

#### Integration with Pots & Winners

The evaluator integrates with the existing `pots/winners.ts` module to auto-determine pot distribution at showdown:

```typescript
import { evaluateShowdown } from '@robkerr/poker-engine'

// Given shown cards + board, determine winner(s) of each pot
const results = evaluateShowdown(handState, shownCards)
// => { pots: [{ id: 'main', winners: ['player-3'], ... }] }
```

### 9.6 Replay Animation

Animate through the hand action-by-action with a play/pause/step control. Each step highlights the current action row and updates community cards + pot in real-time.

---

## Appendix A: Engine API Quick Reference

Key exports from `@robkerr/poker-engine` used by this app:

| Export | Usage |
|--------|-------|
| `createValidationEngine(gameType)` | Create engine instance for a game type |
| `ValidationEngine.getAvailableActions(handState)` | What the active player can do |
| `ValidationEngine.validateAction(action, handState)` | Check if action is legal |
| `ValidationEngine.getBetConstraints(handState)` | Min/max bet/raise amounts |
| `ValidationEngine.isBettingRoundComplete(handState)` | Is the street done? |
| `ValidationEngine.getActivePlayer(handState)` | Who acts next? |
| `ValidationEngine.getDecisionState(handState)` | Full decision context |
| `buildDecisionState({ handState, schema })` | Low-level state builder |
| `replayToPlayerStates(handState, schema)` | Get player states (always, even when betting complete) |
| `initializePlayerStates(players, blinds)` | Set up initial states with blinds posted |
| `applyAction(states, action, currentBet)` | Apply one action to states |
| `resetForNewStreet(states)` | Reset per-round tracking for new street |
| `calculatePot(playerStates)` | Sum of all investments |
| `getNextStreet(street)` | Street sequencing |
| `getExpectedCardCount(street)` | Cards needed for a street |
| `getPositionsForCount(n)` | Position labels for n players |
| `nlheSchema` / `ploSchema` | Game type schema objects |
| `advanceStreet(...)` | Street transition helper |
| `trimBoardToStreet(board, street)` | Trim board for undo |
| `evaluateShowdown(handState, shownCards)` | Auto-determine showdown winners using hand evaluator |
| `GameTypeSchema.evaluator` | Hand evaluator for a game type (NLHE, PLO) |

---

## Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **Whiteboard** | The visual replay view showing community cards + action columns |
| **Hand Record** | Complete persisted record of a poker hand |
| **Action Node** | A single player action within a hand (the unit annotations attach to) |
| **Decision State** | Engine's snapshot of who acts next + what they can do |
| **Hero** | The user/recording player |
| **Street** | Betting round: preflop, flop, turn, river |
| **Running Stack** | Player's chip count after each action |
| **Fold-out** | Hand ends because all but one player folded |
| **Runout** | Remaining board cards dealt when all players are all-in |
