---
name: game-generator
description: Generate playable browser games from user prompts. When someone asks to make, create, or build a game, this skill generates a single-file HTML/JS game, deploys it to GitHub Pages, and shares the playable link. Games include leaderboards so anyone can compete.
metadata: { "openclaw": { "emoji": "🎮", "requires": { "bins": ["git", "node"] } } }
---

# Game Generator Skill

You generate complete, playable HTML5 games using Phaser 3 from user prompts and deploy them to GitHub Pages.

## Configuration

Read `config.json` from this skill's directory to get:
- `GITHUB_REPO_OWNER` — GitHub username
- `GITHUB_REPO_NAME` — repo name (`game-hub`)
- `GITHUB_PAGES_URL` — base URL for published games
- `LEADERBOARD_API_URL` — Cloudflare Worker URL for scores

## Game Design Philosophy

Every game you generate MUST follow these principles. This is what makes games fun.

### Core Loop: Survive & Score

All games are **endless/survival** — the player always eventually loses. There is NO win condition. The player's goal is to survive as long as possible and score as high as possible before dying. This is critical: do NOT create games with win conditions (like "first to 7 points"). Every game ends in death/failure.

### Difficulty Curve

This is the most important part. Games MUST start easy and ramp:

- **0–10 seconds**: Tutorial-easy. The player should feel powerful. Obstacles are slow and sparse. This is the "learn the controls" phase.
- **10–30 seconds**: Warming up. Noticeable but fair increase. A casual player dies here.
- **30–60 seconds**: Challenging. Requires focus and decent reflexes. Average players die here.
- **60–120 seconds**: Hard. Requires skill and some luck. Good players die here.
- **120+ seconds**: Brutal. Speed/density near maximum. Only the best or luckiest survive.

Use **time-based difficulty scaling**, not event-based. Track elapsed seconds since game start and scale parameters from that. Example:

```javascript
const elapsed = (Date.now() - gameStartTime) / 1000;
const difficulty = Math.min(elapsed / 120, 1); // 0 to 1 over 2 minutes
const spawnRate = lerp(0.5, 3.0, difficulty);   // obstacles per second
const speed = lerp(2, 8, difficulty);            // pixels per frame
```

Always use a lerp or easing function — never step functions. The player should not feel sudden jumps in difficulty.

```javascript
function lerp(a, b, t) { return a + (b - a) * t; }
function easeIn(t) { return t * t; }
function easeInOut(t) { return t < 0.5 ? 2*t*t : 1 - Math.pow(-2*t + 2, 2) / 2; }
```

### Duration Targets

- **Minimum run** (worst case): ~15-20 seconds. Even a totally new player should get this far.
- **Average run**: 30-60 seconds.
- **Good run**: 1-2 minutes.
- **Great run**: 2-3+ minutes.

If a playtest in your head ends in under 15 seconds on the easiest settings, the starting difficulty is too high. Dial it down.

### Scoring

- **Primary score = survival time** (in seconds or ticks). Always incrementing while alive.
- **Bonus points** for active play: collecting items (+10-50), destroying enemies (+25-100), combos/streaks (multiplier).
- **Never subtract points.** Score only goes up.
- The final score submitted to the leaderboard MUST be `Math.floor(timeAlive * 10 + bonusPoints)` (or equivalent variable names), and it must be monotonic (never decreases).
- Display the score prominently during gameplay and on the game over screen.

### Game Genres

The genre is open-ended — any game concept works as long as it follows the survive-and-score philosophy above. Be creative and match the user's request. When the concept is vague, here are some proven patterns for inspiration (but don't limit yourself to these):

- Dodgers, runners, arena survivors, flappy-style, wave shooters, stacking/balancing, rhythm/reaction, tower defense, puzzle survival, physics toys, racing against rising danger, territorial control under pressure, etc.

The key constraint is: **the player must always eventually lose**. As long as difficulty ramps over time and there's no win state, any mechanic works.

### Hit Detection & Forgiveness

- Use Phaser's arcade physics for collision. Make **physics bodies slightly smaller** than the visual sprite so near-misses feel fair.
- `sprite.body.setSize(width * 0.8, height * 0.8)` — 20% forgiveness.
- For circles: `sprite.body.setCircle(radius * 0.8)`.
- On death, show a brief death animation (flash, explode, dissolve) — never just freeze. Use `this.cameras.main.shake()` and `this.cameras.main.flash()` for impact.

## Visual Design

### Theme Variety

Do NOT always use the same dark-background-with-neon look. Pick a visual theme that matches the game concept. If no obvious match, randomly select one of these palettes:

**Synthwave/Neon** (only use ~20% of the time)
- BG: `#0a0a2e` → `#1a0533`, Accents: `#ff00ff`, `#00ffff`, `#ff3366`
- Glow effects, grid lines, retro font

**Sunset/Warm**
- BG: `#1a0a2e` → `#ff6b35`, Accents: `#ffd700`, `#ff4757`, `#ff6348`
- Gradient sky, warm particle trails

**Ocean/Cool**
- BG: `#0a1628` → `#1e3a5f`, Accents: `#00b4d8`, `#48cae4`, `#90e0ef`
- Wavy animations, bubble particles

**Forest/Nature**
- BG: `#0a1a0a` → `#1a3a1a`, Accents: `#2ecc71`, `#f1c40f`, `#e67e22`
- Organic shapes, leaf particles

**Candy/Pastel**
- BG: `#2d1b4e` → `#1a1a2e`, Accents: `#ff6b9d`, `#c44dff`, `#ffd93d`, `#6bff6b`
- Rounded shapes, bouncy animations, playful

**Monochrome/Minimal**
- BG: `#111` → `#1a1a1a`, Accents: `#fff`, `#888`, one single accent color
- Clean lines, geometric shapes, high contrast

**Desert/Western**
- BG: `#2d1f0e` → `#5c3d1e`, Accents: `#f4a460`, `#daa520`, `#cd853f`
- Sandy textures, tumbleweeds, cacti

**Cosmic/Space**
- BG: `#050510` → `#0a0a2e`, Accents: `#9b59b6`, `#3498db`, `#e74c3c`, `#f1c40f`
- Star fields, nebula gradients, glowing objects

### Animation Quality

Use Phaser's built-in systems for polish:

- **Particles**: Use `this.add.particles()` with emitter configs for collisions, deaths, score pickups, ambient effects.
- **Screen shake**: `this.cameras.main.shake(duration, intensity)` on impacts and death.
- **Flash effects**: `this.cameras.main.flash(duration, r, g, b)` on damage. `sprite.setTint(0xffffff)` then tween back for hit flash.
- **Tweens**: `this.tweens.add({ targets, props, duration, ease })` for smooth movement, scaling, fading. Use for UI transitions, enemy patterns, score popups.
- **Background movement**: parallax scrolling via `this.add.tileSprite()`, floating particle emitters, gradient backgrounds via graphics objects. The background should never feel dead.

### Typography

Vary fonts via Google Fonts. Don't always use "Press Start 2P". Pick one that matches the theme:
- Retro/arcade: `Press Start 2P`, `VT323`, `Silkscreen`
- Modern/clean: `Orbitron`, `Rajdhani`, `Exo 2`
- Playful: `Fredoka One`, `Bungee`, `Luckiest Guy`
- Elegant: `Cinzel`, `Playfair Display`

Use at most ONE display font (for titles/scores) plus the system default for UI elements.

## Audio Design

Use the Web Audio API to generate sound effects procedurally. This requires NO external files or libraries. Include a sound utility at the top of your script:

```javascript
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

function playSound(type, freq, duration, volume) {
  // Resume context on first user interaction (required by browsers)
  if (audioCtx.state === 'suspended') audioCtx.resume();

  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.connect(gain);
  gain.connect(audioCtx.destination);

  osc.type = type; // 'sine', 'square', 'sawtooth', 'triangle'
  osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
  gain.gain.setValueAtTime(volume || 0.3, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration);

  osc.start();
  osc.stop(audioCtx.currentTime + duration);
}

// Prebuilt sound effects:
function sfxJump()    { playSound('sine', 400, 0.15, 0.2); playSound('sine', 600, 0.1, 0.15); }
function sfxCollect() { playSound('sine', 800, 0.1, 0.25); playSound('sine', 1200, 0.1, 0.15); }
function sfxHit()     { playSound('sawtooth', 150, 0.2, 0.3); playSound('square', 80, 0.3, 0.2); }
function sfxDeath()   { playSound('sawtooth', 300, 0.5, 0.3); playSound('sawtooth', 100, 0.8, 0.2); }
function sfxScore()   { playSound('triangle', 600, 0.1, 0.2); playSound('triangle', 900, 0.15, 0.15); }
```

Customize the frequencies and waveforms to match your game's theme. Always call `audioCtx.resume()` inside a user gesture handler (click/tap to start).

Do NOT play sounds on every frame — only on discrete events (jump, collect, hit, death, score milestone).

## Game Framework: Phaser 3

All games MUST use **Phaser 3** as the game framework. Do not write raw Canvas code. Phaser handles physics, collisions, input, particles, scaling, and the game loop correctly — hand-rolling these is the #1 source of bugs.

Load Phaser via CDN:
```html
<script src="https://cdn.jsdelivr.net/npm/phaser@3.70.0/dist/phaser.min.js"></script>
```

### Why Phaser

- **Arcade physics** handles collision detection and response reliably — no more hand-rolled hitbox math
- **Input manager** handles keyboard, mouse, and touch seamlessly across devices
- **Particle emitter** built in — no manual particle arrays
- **Tweens** for smooth animations, screen shake, flash effects
- **Scene management** for clean start → gameplay → game over transitions
- **Scale manager** for responsive/mobile-friendly sizing
- **Delta-time built in** via `scene.time` and update(time, delta) — no manual dt calculation
- LLMs have extensive training data on Phaser 3; generated code is more reliable

### Phaser Game Skeleton

Every game should follow this structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>{Game Title}</title>
<link href="https://fonts.googleapis.com/css2?family={Font}&display=swap" rel="stylesheet">
<script src="https://cdn.jsdelivr.net/npm/phaser@3.70.0/dist/phaser.min.js"></script>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #000; display: flex; justify-content: center; align-items: center;
         height: 100vh; overflow: hidden; touch-action: none; }
  /* Game over overlay styles */
  #ui-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%;
                display: none; justify-content: center; align-items: center;
                z-index: 10; background: rgba(0,0,0,0.8); font-family: '{Font}', sans-serif; }
  /* ... overlay element styles ... */
</style>
</head>
<body>
<div id="ui-overlay"><!-- Game over form: score display, name input, submit button, leaderboard, replay --></div>
<script>
const GAME_ID = '{slug}';
const LEADERBOARD_API = '{LEADERBOARD_API_URL}';

// --- Leaderboard functions (same as before) ---
// --- Web Audio sound effects (same as before) ---

class MenuScene extends Phaser.Scene {
  constructor() { super('MenuScene'); }
  create() {
    // Title, instructions, "Tap to Start"
    // this.input.once('pointerdown', () => this.scene.start('GameScene'));
    // this.input.keyboard.once('keydown', () => this.scene.start('GameScene'));
  }
}

class GameScene extends Phaser.Scene {
  constructor() { super('GameScene'); }

  create() {
    this.score = 0;
    this.gameStartTime = this.time.now;
    // Resume audio context on first interaction
    if (audioCtx.state === 'suspended') audioCtx.resume();
    // Set up player, obstacles, collisions, input, score display, particles, etc.
  }

  update(time, delta) {
    const elapsed = (time - this.gameStartTime) / 1000;
    const difficulty = Math.min(elapsed / 120, 1);
    // Scale spawn rates, speeds, enemy behavior with difficulty
    // Update score: this.score += delta / 100; (time-based)
    // Check death conditions
  }

  gameOver() {
    // Death animation, camera shake, particles
    this.cameras.main.shake(200, 0.01);
    sfxDeath();
    // Brief delay then show overlay
    this.time.delayedCall(500, () => {
      this.scene.pause();
      showGameOverUI(Math.floor(this.score));
    });
  }
}

const config = {
  type: Phaser.AUTO,
  width: 800,
  height: 600,
  backgroundColor: '{bg_color}',
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 0 }, debug: false }
  },
  scale: {
    mode: Phaser.Scale.FIT,
    autoCenter: Phaser.Scale.CENTER_BOTH,
  },
  scene: [MenuScene, GameScene]
};

const game = new Phaser.Game(config);

// --- Game over UI logic (show overlay, handle submit, show leaderboard, replay) ---
function showGameOverUI(finalScore) { /* ... */ }
</script>
</body>
</html>
```

### Asset Pipeline & Visual Generation

Games MAY use external image files when they improve quality.

Use this priority order:
1. User-supplied assets (Discord attachments/URLs) when provided
2. Agent-sourced assets from reputable, permissive sources when browsing is available
3. Procedural textures as fallback when assets are unavailable or unreliable

If browsing/image search is unavailable, skip step 2 and go directly to procedural fallback.

When using external assets:
- Prefer direct HTTPS URLs and verify usage rights (CC0/public domain/user-owned/licensed).
- Add a short source comment near the top of the script for each external asset URL.
- Load in `preload()` using Phaser's loader (`this.load.image`, `this.load.spritesheet`).
- Add load-error fallback to generated textures so the game still works if an asset URL fails.
- Keep gameplay readability first: strong contrast, consistent sprite scale, uncluttered hit areas.

When using procedural art:
- Do NOT use plain `fillRect` boxes for player/enemy sprites unless the concept is intentionally blocky.
- Build each core gameplay sprite with at least 3 visual layers (base, outline/shadow, highlight/detail).
- Generate at higher resolution (`96x96` or `128x128`) and scale down in-game for cleaner results.
- Vary silhouette language across entities (for example: circular player, triangular projectile, irregular enemy).
- Add subtle motion polish (idle bob, rotate, squash/stretch tween) so entities feel alive.

```javascript
// preload(): load external asset with procedural fallback
this.load.image('enemySprite', 'https://example.com/enemy.png');
this.load.on('loaderror', (file) => {
  if (file.key === 'enemySprite') {
    const g = this.add.graphics();
    g.fillStyle(0x7a1f2a, 1);
    g.fillCircle(48, 48, 34);
    g.lineStyle(6, 0xffd166, 1);
    g.strokeCircle(48, 48, 34);
    g.fillStyle(0xffffff, 0.35);
    g.fillCircle(58, 36, 10);
    g.generateTexture('enemySprite', 96, 96);
    g.destroy();
  }
});
```

Use `generateTexture()` for physics/particles and use text objects/emoji for decoration only.

### Additional CDN Dependencies

Beyond Phaser, you may add other CDN libraries when they genuinely improve the game. Always pin versions. Examples:
- **Howler.js** for music/complex audio: `https://cdn.jsdelivr.net/npm/howler@2.2.4/dist/howler.min.js`

Games are static HTML files that load Phaser via CDN — no build step is needed. Just push the HTML file and GitHub Pages serves it.

## Workflow

### 1. Expand short prompts into a complete concept (MANDATORY)

Assume user prompts may be short. Do **NOT** ask follow-up questions. Infer missing details and produce a complete game concept before coding.

If the concept is vague ("make a fun game"), pick a fitting survival genre from the patterns above.

Define all of the following up front (reconciling with any explicit user constraints):
1. **Controls**: keyboard + touch
2. **Core loop + death condition**
3. **2–3 hazards/enemies** with distinct behaviors
4. **1–2 pickups/upgrades** and at least one **risk/reward** mechanic
5. **Scoring**: `score = Math.floor(timeAlive * 10 + bonusPoints)` and score must be monotonic
6. **Difficulty ramp parameters** (`spawnRate`, `speed`, `density`) using smooth time-based lerp/ease
7. **Visual theme**: palette + font choice + background motion
8. **Audio SFX style** that matches the theme

Always inspect the request for asset inputs (Discord attachments, image URLs, sprite sheets) and treat them as top-priority art direction constraints.

### 2. Write a DESIGN HEADER, then implement

At the top of the `<script>` tag, write a short `DESIGN HEADER` comment summarizing the decisions from Step 1. Keep it concise and concrete.

Then implement the game fully. Do not leave TODO/FIXME placeholders.

### 3. Generate a slug

Create a URL-safe slug: `{game-name}-{4-char-random-hex}`. Examples: `asteroid-dodge-a1b2`, `lava-run-f3e9`

### 4. Generate the game HTML

Create a **complete single-file HTML game** using Phaser 3, following the skeleton in the "Game Framework" section above and all requirements below.

**Technical requirements:**
- Phaser 3 loaded via CDN `<script>` tag — this is the game framework, not raw Canvas
- All game CSS and JS inline in a single HTML file
- External assets are allowed via URL (user-provided or agent-sourced) with reliable fallback
- Google Fonts via `<link>` for typography
- Web Audio API for procedural sound effects
- Phaser's `Scale.FIT` + `CENTER_BOTH` for responsive mobile scaling
- Phaser's built-in input handling for keyboard + pointer/touch
- Use Phaser's `update(time, delta)` for delta-time scaling — `delta` is milliseconds since last frame. Normalize: `const dt = delta / 16.667;` and multiply all movement by `dt`.

**Game structure (as Phaser scenes):**
- **MenuScene**: Title, brief instructions, "Tap to Start" / "Press any key". Transition to GameScene on input.
- **GameScene**: Core gameplay with visible score counter. Handles difficulty ramping, spawning, physics, collisions, player input. On death: animation + delay, then pause scene and show HTML overlay.
- Game over UI is an **HTML overlay** (not a Phaser scene) so the name input and leaderboard form work reliably.
- On "Play Again": hide overlay, `scene.restart()` the GameScene (Phaser handles full state reset).

**Bake these constants at the top of the `<script>` tag:**
```javascript
const GAME_ID = '{slug}';
const LEADERBOARD_API = '{LEADERBOARD_API_URL from config}';
```

**Include these leaderboard functions:**
```javascript
async function submitScore(playerName, score) {
  try {
    const res = await fetch(`${LEADERBOARD_API}/api/scores`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ gameId: GAME_ID, playerName, score })
    });
    return await res.json();
  } catch (e) {
    console.warn('Leaderboard unavailable:', e);
    return null;
  }
}

async function getLeaderboard(limit = 10) {
  try {
    const res = await fetch(`${LEADERBOARD_API}/api/scores?gameId=${GAME_ID}&limit=${limit}`);
    const data = await res.json();
    return data.scores || [];
  } catch (e) {
    console.warn('Leaderboard unavailable:', e);
    return [];
  }
}
```

**Game over flow (use this exact sequence):**
1. Play death sound + death animation (explosion, fade, etc.)
2. After a brief delay (~500ms), show the game over overlay
3. Display final score prominently
4. Show an HTML `<input>` field (NOT `prompt()`) for the player's name
5. Show a "Submit Score" button
6. On submit: call `submitScore()`, then call `getLeaderboard()` and display top 10
7. Show a "Play Again" button that fully resets the game

### 5. Deploy to GitHub Pages (lock + retry required)

Use a serialized deploy flow so multiple agents on the same machine do not race each other.

```bash
# Serialize deploys across local agents
LOCKFILE=/tmp/openclaw-game-hub.deploy.lock
exec 9>"$LOCKFILE"
flock -w 180 9 || { echo "Deploy lock timeout" >&2; exit 1; }

# Use the game-hub deploy key (public key in GitHub: ~/.ssh/id_ed25519.pub)
# Git uses the matching private key file locally: ~/.ssh/id_ed25519
test -f ~/.ssh/id_ed25519 || { echo "Missing ~/.ssh/id_ed25519 deploy key" >&2; exit 1; }
export GIT_SSH_COMMAND='ssh -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new'

# Create a temp directory
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT

# Clone the hub repo fresh
git clone "git@github.com:{GITHUB_REPO_OWNER}/{GITHUB_REPO_NAME}.git" "$TMPDIR/repo"
cd "$TMPDIR/repo"

# Create the game directory + copy game file
mkdir -p "games/{slug}"
cp /path/to/generated/game.html "games/{slug}/index.html"

# Update games.json with an UPSERT by slug (avoid duplicates)
node -e '
const fs = require("fs");
const path = "games.json";
const slug = "{slug}";
const entry = {
  slug,
  title: "{Game Title}",
  description: "{Brief description}",
  creator: "{discord username who requested it}",
  date: "{ISO date string}"
};
const data = JSON.parse(fs.readFileSync(path, "utf8"));
const next = [...data.filter(g => g.slug !== slug), entry];
fs.writeFileSync(path, JSON.stringify(next, null, 2) + "\n");
'

# Commit
git add "games/{slug}/index.html" games.json
git commit -m "Add game: {Game Title}"

# Push with automatic rebase/retry for remote races
for attempt in 1 2 3 4 5; do
  if git push origin main; then
    echo "Deploy succeeded"
    break
  fi

  if [ "$attempt" -eq 5 ]; then
    echo "Deploy failed after retries" >&2
    exit 1
  fi

  git pull --rebase origin main
  sleep $((attempt * 2))
done
```

GitHub Pages should be enabled on the repo (Settings > Pages > Deploy from branch: `main`). Games are static HTML files with CDN dependencies — no build step needed.

### 6. Reply in Discord

Send a message with:
- The game title
- A brief description of how to play
- The playable link: `{GITHUB_PAGES_URL}/games/{slug}/`
- Mention the leaderboard

## Quality Checklist

Before deploying, mentally playtest the game by reading through the code. Verify:

- [ ] **Starts easy**: First 10 seconds are trivially easy at the initial parameter values
- [ ] **Difficulty ramps smoothly**: Parameters scale with elapsed time via lerp, not steps
- [ ] **Player always eventually dies**: No win condition. Game ends only on death/failure.
- [ ] **Score formula is correct and monotonic**: Uses `Math.floor(timeAlive * 10 + bonusPoints)` and never decreases
- [ ] **30s minimum run is achievable**: A new player can survive at least 15-20 seconds
- [ ] **Has a start screen**: Title, instructions, "tap to start"
- [ ] **Has actual gameplay**: Not just a static screen
- [ ] **Enemy variety exists**: At least 2 hazards/enemies with distinct behaviors
- [ ] **Risk/reward exists**: At least 1 pickup/upgrade has a meaningful tradeoff
- [ ] **DESIGN HEADER present**: Top-of-script comment summarizes controls, loop/death, hazards, pickups/risk-reward, scoring, difficulty, visual, and audio choices
- [ ] **Game over triggers correctly**: Death condition works reliably
- [ ] **Death feels impactful**: Sound + visual feedback on death
- [ ] **Leaderboard input is HTML** element, not `prompt()`
- [ ] **Leaderboard submit/display works** with try/catch fallback
- [ ] **Play again fully resets**: Score, difficulty, positions, timers, particles — all reset
- [ ] **Touch controls work** alongside keyboard (Phaser input handles this)
- [ ] **Delta-time used**: Movement uses `delta` from Phaser's `update(time, delta)`
- [ ] **Hitboxes are forgiving**: Physics bodies slightly smaller than visual sprites
- [ ] **Visually distinct theme**: Not the default neon-on-dark unless it fits the concept
- [ ] **Art direction is explicit**: Shape language + motion style are defined in code comments before implementation
- [ ] **Core sprites are not default boxes**: Player/enemy silhouettes are intentionally designed
- [ ] **Has sound effects**: At least: start, action (jump/shoot/collect), death
- [ ] **Asset strategy is resilient**: External assets have license clarity and load-failure fallback
- [ ] **Uses Phaser 3**: Game is built with Phaser, not raw Canvas
- [ ] **All code in single HTML file**: Except Phaser CDN, optional CDN libs, and Google Fonts
- [ ] **AudioContext resumes on user gesture**: First click/tap calls `audioCtx.resume()`

## Common Pitfalls — Avoid These

1. **Starting too fast/hard**: The #1 issue. If initial speed or spawn rate makes the first 5 seconds challenging, it's too high. Cut starting values in half, then test again.
2. **AI opponents that are too good**: If there's an AI opponent, it MUST be beatable at the start and should make mistakes. Scale AI skill with difficulty, not from the start.
3. **Linear difficulty**: `difficulty = time * constant` ramps too fast early and too slow late. Use `Math.pow(t, 0.5)` or `1 - Math.exp(-t * rate)` for a curve that's gentle at first and intense later.
4. **Missing state reset on replay**: Use `this.scene.restart()` — it resets the scene cleanly. But verify any module-level variables outside the scene class also get reset.
5. **Game loop continues after death**: Call `this.scene.pause()` before showing game over UI. The scene stops updating.
6. **Objects spawning on top of the player**: Add a safe zone around the player's starting position. New obstacles should spawn off-screen or away from the player.
7. **Score only from pickups/kills**: If the player can score 0 by hiding in a corner, the scoring is broken. Time-alive should always contribute to score.
8. **Not using Phaser Scale manager**: Always set `scale: { mode: Phaser.Scale.FIT, autoCenter: Phaser.Scale.CENTER_BOTH }` in the config. This handles mobile scaling automatically.
9. **Touch controls not working**: Phaser's input manager handles pointer events (mouse + touch) uniformly. Use `this.input.on('pointerdown')` etc., not raw DOM touch events. For virtual joysticks or drag controls, use `this.input.on('pointermove')`.
10. **No audio context resume**: Browsers block audio until a user gesture. Resume the AudioContext in your MenuScene's start handler.
11. **Boxy placeholder art shipped to production**: If player/enemy are simple squares with no silhouette design, stop and improve art direction before deploy.
12. **Unverified random image usage**: Do not pull arbitrary web images without checking rights and adding a fallback for load failures.
