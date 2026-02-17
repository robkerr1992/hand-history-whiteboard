# Card Interface Architecture (PHP)

A minimal, game-agnostic card abstraction layer. The `Card` interface knows nothing about rank, suit, or any game-specific concept. Concrete implementations define what a card *is* for their game type.

---

## Core Interfaces

### Card

The fundamental unit. A card can identify itself and be compared. Nothing more.

```php
<?php

namespace Cards\Contracts;

interface Card
{
    /**
     * Human-readable representation of this card.
     */
    public function toString(): string;

    /**
     * Whether this card is the same as another.
     */
    public function equals(Card $other): bool;

    /**
     * Unique identifier within a deck (e.g., "As", "Kh", "draw4-red").
     */
    public function id(): string;
}
```

### CardFactory

Produces a complete set of cards for a game type. Different games produce different cards.

```php
<?php

namespace Cards\Contracts;

interface CardFactory
{
    /**
     * Create the full set of cards for this game type.
     *
     * @return Card[]
     */
    public function createDeck(): array;
}
```

### CardFactoryRegistry

Maps game types to their factories. The single lookup point for "give me the cards for this game."

```php
<?php

namespace Cards\Contracts;

interface CardFactoryRegistry
{
    /**
     * Register a factory for a game type.
     */
    public function register(string $gameType, CardFactory $factory): void;

    /**
     * Retrieve the factory for a game type.
     *
     * @throws \InvalidArgumentException If game type is not registered.
     */
    public function get(string $gameType): CardFactory;

    /**
     * Check if a game type has a registered factory.
     */
    public function has(string $gameType): bool;
}
```

---

## Core Classes

### Deck

A generic deck that works with any `Card` implementation. Holds cards, shuffles, deals.

```php
<?php

namespace Cards;

use Cards\Contracts\Card;

class Deck
{
    /** @var Card[] */
    private array $cards;

    /**
     * @param Card[] $cards
     */
    public function __construct(array $cards)
    {
        $this->cards = $cards;
    }

    public function shuffle(): void
    {
        shuffle($this->cards);
    }

    /**
     * Draw N cards from the top of the deck.
     *
     * @return Card[]
     * @throws \UnderflowException If not enough cards remain.
     */
    public function draw(int $count): array
    {
        if ($count > $this->remaining()) {
            throw new \UnderflowException(
                "Cannot draw {$count} cards, only {$this->remaining()} remaining."
            );
        }

        return array_splice($this->cards, 0, $count);
    }

    public function remaining(): int
    {
        return count($this->cards);
    }
}
```

### InMemoryCardFactoryRegistry

Simple in-memory implementation of the registry.

```php
<?php

namespace Cards;

use Cards\Contracts\CardFactory;
use Cards\Contracts\CardFactoryRegistry;

class InMemoryCardFactoryRegistry implements CardFactoryRegistry
{
    /** @var array<string, CardFactory> */
    private array $factories = [];

    public function register(string $gameType, CardFactory $factory): void
    {
        $this->factories[$gameType] = $factory;
    }

    public function get(string $gameType): CardFactory
    {
        if (!$this->has($gameType)) {
            throw new \InvalidArgumentException(
                "No card factory registered for game type: {$gameType}"
            );
        }

        return $this->factories[$gameType];
    }

    public function has(string $gameType): bool
    {
        return isset($this->factories[$gameType]);
    }
}
```

---

## Implementation: Standard Playing Cards

This is the concrete implementation for any game that uses the standard 52-card deck (NLHE, PLO, Blackjack, etc.).

### Rank & Suit Enums

```php
<?php

namespace Cards\PlayingCards;

enum Rank: string
{
    case Two = '2';
    case Three = '3';
    case Four = '4';
    case Five = '5';
    case Six = '6';
    case Seven = '7';
    case Eight = '8';
    case Nine = '9';
    case Ten = 'T';
    case Jack = 'J';
    case Queen = 'Q';
    case King = 'K';
    case Ace = 'A';
}
```

```php
<?php

namespace Cards\PlayingCards;

enum Suit: string
{
    case Spades = 's';
    case Hearts = 'h';
    case Diamonds = 'd';
    case Clubs = 'c';

    public function symbol(): string
    {
        return match ($this) {
            self::Spades => '♠',
            self::Hearts => '♥',
            self::Diamonds => '♦',
            self::Clubs => '♣',
        };
    }
}
```

### PlayingCard

A standard playing card. Implements `Card` and adds rank + suit.

```php
<?php

namespace Cards\PlayingCards;

use Cards\Contracts\Card;

class PlayingCard implements Card
{
    public function __construct(
        public readonly Rank $rank,
        public readonly Suit $suit,
    ) {}

    public function toString(): string
    {
        return $this->rank->value . $this->suit->symbol();
    }

    public function equals(Card $other): bool
    {
        if (!$other instanceof self) {
            return false;
        }

        return $this->rank === $other->rank
            && $this->suit === $other->suit;
    }

    public function id(): string
    {
        return $this->rank->value . $this->suit->value;
    }
}
```

### PlayingCardFactory

Produces a standard 52-card deck.

```php
<?php

namespace Cards\PlayingCards;

use Cards\Contracts\CardFactory;

class PlayingCardFactory implements CardFactory
{
    public function createDeck(): array
    {
        $cards = [];

        foreach (Suit::cases() as $suit) {
            foreach (Rank::cases() as $rank) {
                $cards[] = new PlayingCard($rank, $suit);
            }
        }

        return $cards;
    }
}
```

### ShortDeckFactory

36-card deck for Short Deck Hold'em (6+ only). Same `PlayingCard`, different factory.

```php
<?php

namespace Cards\PlayingCards;

use Cards\Contracts\CardFactory;

class ShortDeckFactory implements CardFactory
{
    private const EXCLUDED_RANKS = [
        Rank::Two,
        Rank::Three,
        Rank::Four,
        Rank::Five,
    ];

    public function createDeck(): array
    {
        $cards = [];

        foreach (Suit::cases() as $suit) {
            foreach (Rank::cases() as $rank) {
                if (in_array($rank, self::EXCLUDED_RANKS, true)) {
                    continue;
                }

                $cards[] = new PlayingCard($rank, $suit);
            }
        }

        return $cards;
    }
}
```

---

## Wiring It Together

```php
<?php

use Cards\InMemoryCardFactoryRegistry;
use Cards\Deck;
use Cards\PlayingCards\PlayingCardFactory;
use Cards\PlayingCards\ShortDeckFactory;

// Register game types
$registry = new InMemoryCardFactoryRegistry();
$registry->register('nlhe', new PlayingCardFactory());
$registry->register('plo', new PlayingCardFactory());
$registry->register('short-deck', new ShortDeckFactory());

// Create a deck for a game
$factory = $registry->get('nlhe');
$deck = new Deck($factory->createDeck());
$deck->shuffle();

$holeCards = $deck->draw(2);  // [PlayingCard, PlayingCard]
$flop = $deck->draw(3);
$turn = $deck->draw(1);
$river = $deck->draw(1);
```

---

## Directory Structure

```
src/Cards/
├── Contracts/
│   ├── Card.php
│   ├── CardFactory.php
│   └── CardFactoryRegistry.php
├── Deck.php
├── InMemoryCardFactoryRegistry.php
├── PlayingCards/
│   ├── Rank.php
│   ├── Suit.php
│   ├── PlayingCard.php
│   ├── PlayingCardFactory.php
│   └── ShortDeckFactory.php
├── Mtg/
│   ├── MtgColor.php
│   ├── MtgCardType.php
│   ├── MtgCard.php
│   └── MtgDeckFactory.php
└── Uno/
    ├── UnoColor.php
    ├── UnoValue.php
    ├── UnoCard.php
    └── UnoCardFactory.php
```

---

## Extensibility Examples

### Uno

Completely different card shape — color + value instead of rank + suit.

```php
class UnoCard implements Card
{
    public function __construct(
        public readonly UnoColor $color,
        public readonly UnoValue $value,
    ) {}

    // ... implements toString(), equals(), id()
}

class UnoCardFactory implements CardFactory
{
    public function createDeck(): array
    {
        // 108 cards: number cards, skip, reverse, draw two, wild, wild draw four
    }
}

$registry->register('uno', new UnoCardFactory());
```

### Magic: The Gathering

MTG pushes the design harder — cards have *many* properties, some optional, some unique to card types. But the `Card` interface doesn't care. An MTG card is just another implementation.

```php
<?php

namespace Cards\Mtg;

enum MtgColor: string
{
    case White = 'W';
    case Blue = 'U';
    case Black = 'B';
    case Red = 'R';
    case Green = 'G';
    case Colorless = 'C';
}

enum MtgCardType: string
{
    case Creature = 'Creature';
    case Instant = 'Instant';
    case Sorcery = 'Sorcery';
    case Enchantment = 'Enchantment';
    case Artifact = 'Artifact';
    case Planeswalker = 'Planeswalker';
    case Land = 'Land';
}
```

```php
<?php

namespace Cards\Mtg;

use Cards\Contracts\Card;

class MtgCard implements Card
{
    /**
     * @param MtgColor[]     $colors
     * @param MtgCardType[]  $types
     * @param string[]       $subtypes  e.g., ['Elf', 'Warrior']
     */
    public function __construct(
        public readonly string $name,
        public readonly int $manaCost,
        public readonly array $colors,
        public readonly array $types,
        public readonly array $subtypes = [],
        public readonly ?int $power = null,
        public readonly ?int $toughness = null,
        public readonly ?string $rulesText = null,
        public readonly string $set = '',
        public readonly string $rarity = 'common',
    ) {}

    public function toString(): string
    {
        $base = "{$this->name} ({$this->manaCost})";

        if ($this->isCreature()) {
            return "{$base} {$this->power}/{$this->toughness}";
        }

        return $base;
    }

    public function equals(Card $other): bool
    {
        if (!$other instanceof self) {
            return false;
        }

        // In MTG, same name + same set = same card
        return $this->name === $other->name
            && $this->set === $other->set;
    }

    public function id(): string
    {
        // Unique within a set
        return strtolower(str_replace(' ', '-', $this->name)) . '-' . $this->set;
    }

    public function isCreature(): bool
    {
        return in_array(MtgCardType::Creature, $this->types, true);
    }

    public function isLand(): bool
    {
        return in_array(MtgCardType::Land, $this->types, true);
    }

    /**
     * @return MtgColor[]
     */
    public function colorIdentity(): array
    {
        return $this->colors;
    }
}
```

```php
<?php

namespace Cards\Mtg;

use Cards\Contracts\CardFactory;

class MtgDeckFactory implements CardFactory
{
    /** @var MtgCard[] */
    private array $cards;

    /**
     * MTG decks aren't generated — they're constructed from a card pool.
     * The factory accepts a decklist (card definitions) and produces the deck.
     *
     * @param array<array{card: MtgCard, quantity: int}> $decklist
     */
    public function __construct(
        private readonly array $decklist,
    ) {}

    public function createDeck(): array
    {
        $cards = [];

        foreach ($this->decklist as $entry) {
            for ($i = 0; $i < $entry['quantity']; $i++) {
                $cards[] = $entry['card'];
            }
        }

        return $cards;
    }
}
```

```php
<?php

// Wiring: build an MTG deck from a decklist

use Cards\Mtg\{MtgCard, MtgCardType, MtgColor, MtgDeckFactory};
use Cards\Deck;

$lightning = new MtgCard(
    name: 'Lightning Bolt',
    manaCost: 1,
    colors: [MtgColor::Red],
    types: [MtgCardType::Instant],
    rulesText: 'Lightning Bolt deals 3 damage to any target.',
    set: 'M21',
    rarity: 'uncommon',
);

$goblinGuide = new MtgCard(
    name: 'Goblin Guide',
    manaCost: 1,
    colors: [MtgColor::Red],
    types: [MtgCardType::Creature],
    subtypes: ['Goblin', 'Scout'],
    power: 2,
    toughness: 2,
    rulesText: 'Haste. Whenever Goblin Guide attacks, defending player reveals the top card of their library.',
    set: 'ZEN',
    rarity: 'rare',
);

$mountain = new MtgCard(
    name: 'Mountain',
    manaCost: 0,
    colors: [MtgColor::Red],
    types: [MtgCardType::Land],
    set: 'M21',
);

$factory = new MtgDeckFactory([
    ['card' => $lightning,   'quantity' => 4],
    ['card' => $goblinGuide, 'quantity' => 4],
    ['card' => $mountain,    'quantity' => 20],
    // ... rest of the 60-card deck
]);

$registry->register('mtg', $factory);

$deck = new Deck($factory->createDeck());
$deck->shuffle();
$hand = $deck->draw(7);  // Opening hand
```

---

## Design Principles

1. **Card is data, the game is logic.** They never bleed into each other.
2. **The Deck doesn't know how it was created.** It receives cards and manages them. Generation is the Factory's job.
3. **The Factory is the strategy.** Different games = different factories. Same interface, completely different creation logic (algorithmic for poker, user-constructed for MTG).
4. **The Registry is the lookup.** One place to ask "give me the cards for this game type."
5. **Extend by implementing, not modifying.** To add a new card game, implement `Card` and `CardFactory`. The core never changes.

### The Five Pieces

| Piece | Role | Changes when? |
|---|---|---|
| `Card` | Identity contract | Never |
| `CardFactory` | Creation contract | Never |
| `CardFactoryRegistry` | Lookup contract | Never |
| `Deck` | Card management | Never |
| `XyzCard + XyzCardFactory` | Game-specific implementation | New game type added |
