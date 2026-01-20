# Hungry Critters

![License](https://img.shields.io/badge/license-MIT-blue) ![HTML5](https://img.shields.io/badge/HTML5-E34F26?logo=html5&logoColor=white) ![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?logo=javascript&logoColor=black) ![Canvas](https://img.shields.io/badge/API-Canvas2D-orange)

A browser-based Snake-inspired arcade game where you control a hungry critter that grows longer with every food item it eats. Avoid the walls, avoid your own tail, and don't let the critters collide. Built with the HTML5 Canvas API and vanilla JavaScript — no frameworks, no build tools.

---

## Table of Contents

- [Gameplay](#gameplay)
- [Features](#features)
- [Controls](#controls)
- [Game Modes](#game-modes)
- [Scoring](#scoring)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Game Architecture](#game-architecture)
- [AI Critters](#ai-critters)
- [Configuration](#configuration)
- [Browser Compatibility](#browser-compatibility)
- [Contributing](#contributing)
- [License](#license)

---

## Gameplay

You control a critter on a grid. Food items appear at random positions. Eat a food item and your critter grows one segment longer. The longer you get, the harder it is to avoid yourself.

AI-controlled critters also roam the grid competing for food. They avoid walls and use basic pathfinding to chase the nearest food item. If your critter's head collides with a wall, the edge of the grid, or any critter's body segment (including your own), the round ends.

---

## Features

- Classic snake-style grid movement
- AI-controlled competing critters with BFS pathfinding
- Multiple food types with different point values and effects
- Wraparound walls mode (critter exits one side and enters the opposite)
- Speed increase over time — game gets faster as your score grows
- Up to 4 player critters in local multiplayer (keyboard split)
- Power-up items: Speed Boost, Shield, Magnet (auto-collect nearby food), Freeze (halts all AI critters)
- High score tracking via `localStorage`
- Grid sizes: Small (15x15), Medium (20x20), Large (30x30)
- Animated sprites for each critter (head/body/tail/turn segments)
- Sound effects for eating, collision, and power-up pickup
- Pause support and configurable speed settings

---

## Controls

### Single Player

| Key | Action |
|---|---|
| Arrow Up / W | Move up |
| Arrow Down / S | Move down |
| Arrow Left / A | Move left |
| Arrow Right / D | Move right |
| P / Escape | Pause / Resume |
| R | Restart after game over |
| M | Mute / Unmute |

### Local Multiplayer (Player 2)

| Key | Action |
|---|---|
| Arrow Up | Move up |
| Arrow Down | Move down |
| Arrow Left | Move left |
| Arrow Right | Move right |

### Mobile

| Gesture | Action |
|---|---|
| Swipe Up | Move up |
| Swipe Down | Move down |
| Swipe Left | Move left |
| Swipe Right | Move right |

---

## Game Modes

### Classic
One player, AI critters compete for food. Standard walls — hitting the edge ends the round.

### Survival
No AI critters. Just you and an increasingly tight space as your critter grows. Speed increases faster than Classic.

### Multiplayer (up to 4 players)
Local split-screen on a single keyboard. Each player controls one critter with a distinct key set. Last critter alive wins the round. Play best of 3 or best of 5.

### Wraparound
Same as Classic but walls are passable — exit the right edge and reappear on the left. Makes avoiding collisions harder since the critter never naturally stops at the boundary.

---

## Scoring

| Action | Points |
|---|---|
| Eating standard food | +10 |
| Eating rare food (golden) | +50 |
| Eating bonus food (flashing) | +100 |
| Eating a power-up | +25 |
| Outlasting an AI critter | +200 |
| Every 10 food items eaten | +150 bonus |

Score multiplier: increases 0.25x for each AI critter still alive when you eat. More competition = higher multiplier.

---

## Getting Started

### Open directly in browser

```bash
git clone https://github.com/Salmanahmed1078/Hungry-Critters.git
cd Hungry-Critters
open index.html
```

No install, no build step. Works fully offline.

### Local server (for audio)

```bash
# Python
python3 -m http.server 8080

# Node.js
npx serve .

# VS Code Live Server: right-click index.html → Open with Live Server
```

Open [http://localhost:8080](http://localhost:8080).

---

## Project Structure

```
Hungry-Critters/
├── index.html              # Canvas element and game UI
├── css/
│   ├── style.css           # HUD, score panels, menus
│   └── responsive.css      # Mobile layout
├── js/
│   ├── main.js             # Entry point — bootstraps game loop
│   ├── game.js             # Game state machine and loop
│   ├── critter.js          # Critter movement, growth, collision
│   ├── player.js           # Human player input handling
│   ├── ai.js               # AI critter BFS pathfinding
│   ├── food.js             # Food types and random spawning
│   ├── powerups.js         # Power-up types and timed effects
│   ├── grid.js             # Grid data structure and helpers
│   ├── renderer.js         # Canvas 2D drawing layer
│   ├── input.js            # Keyboard and touch input handler
│   ├── audio.js            # Sound effects
│   ├── score.js            # Score tracking and localStorage
│   └── config.js           # Grid size, speed, difficulty constants
├── assets/
│   ├── sprites/            # Critter and food sprite sheets
│   └── sounds/             # .ogg and .mp3 sound effects
└── README.md
```

---

## Game Architecture

### Game loop

The game runs on a tick-based update (not frame-rate dependent). The critter moves one tile per tick, and the tick interval decreases as the score grows:

```js
function startTicker() {
  tickInterval = setInterval(() => {
    game.update();
    renderer.draw(game.state);
  }, CONFIG.TICK_INTERVAL_MS);
}
```

Each tick: all critters move one step in their current direction, collision checks run, food consumption is evaluated, AI critters re-calculate their next direction.

### State machine

```
MENU → MODE_SELECT → PLAYING → PAUSED → PLAYING
                        │
                        └── GAME_OVER → SCORE_SCREEN → MENU
```

### Critter data structure

Each critter is represented as a doubly-ended queue (deque) of grid positions:

```js
class Critter {
  constructor(start, direction) {
    this.segments = [start];   // [head, body..., tail]
    this.direction = direction;
    this.pendingGrowth = 0;
  }

  move(newHead) {
    this.segments.unshift(newHead);
    if (this.pendingGrowth > 0) {
      this.pendingGrowth--;
    } else {
      this.segments.pop();     // remove tail
    }
  }

  eat(food) {
    this.pendingGrowth += food.growthAmount;
  }
}
```

Adding growth to `pendingGrowth` and deferring the tail removal keeps the critter at the right length without rebuilding the array.

---

## AI Critters

AI critters use Breadth-First Search (BFS) to find the shortest path to the nearest food item each tick. They avoid tiles occupied by any critter body segment.

```
1. Find all food items on the grid
2. Run BFS from AI head to each food item
3. Choose the food with the shortest path
4. Set next direction = first step on that path
5. If no path exists (critter is trapped), fallback: try to spiral outward
```

AI critters do not look ahead for future self-collision — they are reactive, not predictive. This makes them beatable with good positioning.

The number of AI critters and their re-pathfinding frequency are configurable in `config.js`.

---

## Configuration

```js
const CONFIG = {
  GRID_SIZE: 20,              // Grid dimensions (square)
  CELL_SIZE: 24,              // Pixels per grid cell
  TICK_INTERVAL_MS: 150,      // Starting tick rate (ms per move)
  MIN_TICK_INTERVAL_MS: 60,   // Fastest possible tick rate
  SPEED_INCREMENT: 5,         // Reduce interval by this ms per 50 points
  AI_CRITTER_COUNT: 2,        // AI critters on the grid
  AI_REPATH_EVERY: 3,         // AI re-plans path every N ticks
  FOOD_SPAWN_RATE: 0.02,      // Probability of spawning food each tick
  RARE_FOOD_CHANCE: 0.1,      // Fraction of food that is rare (golden)
  POWER_UP_CHANCE: 0.05,      // Fraction of items that are power-ups
  WRAPAROUND: false,          // Passable walls
  SOUND_ENABLED: true,
};
```

---

## Browser Compatibility

| Browser | Minimum Version | Notes |
|---|---|---|
| Chrome | 60+ | Full support |
| Firefox | 55+ | Full support |
| Safari | 12+ | Full support |
| Edge | 79+ | Full support |
| iOS Safari | 13+ | Touch swipe controls |
| Android Chrome | 70+ | Touch swipe controls |

Requires HTML5 Canvas 2D and `setInterval` / `requestAnimationFrame`.

---

## Contributing

1. Fork the repository
2. Work in a feature branch: `git checkout -b feat/your-feature`
3. Test in at least two browsers
4. For AI changes, verify the AI does not get trapped in common grid configurations
5. Open a pull request describing what changed

No external dependencies — keep the zero-dependency principle.

---

## License

MIT © Salman Ahmed
