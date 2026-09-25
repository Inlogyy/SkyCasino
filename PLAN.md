# JerryRoulette — Implementation Plan

Client-side Fabric mod for Minecraft 26.2 that turns opening a Jerry Box on Hypixel SkyBlock into a spinning
prize-wheel animation. **Hypixel decides the reward. The mod reads the result and animates toward it.**

The reward format is known from your screenshots (§15). Items marked ⏳ are small details that one recorded opening
will confirm; the design stays safe without them.

---

## 1. Environment (verified)

| | |
|---|---|
| Minecraft | 26.2 (unobfuscated, official Mojang names, no mappings) |
| Fabric Loader / API | 0.19.5 / 0.161.0+26.2 |
| Loom / Gradle / Java | 1.18.2 / 9.7.1 / 25 (built with Prism's bundled Java 25) |
| Dependencies | **Fabric API only.** Nothing else is required. |
| Install target | `PrismLauncher/instances/Mod Testing/minecraft/mods/jerryroulette-1.0.0.jar` |

APIs in use (all checked against the decompiled 26.2 sources):
`GuiGraphicsExtractor` (replaces GuiGraphics), custom `GuiElementRenderState` for wedge geometry, `HudElementRegistry`,
`ScreenEvents` / `ScreenKeyboardEvents` / `ScreenMouseEvents`, `ClientReceiveMessageEvents`, `KeyMappingHelper`,
`ClientCommands`, `UseItemCallback`, `SimpleSoundInstance.forUI`, `MultiPlayerGameMode.handleContainerInput`
(clicks; `ContainerInput` replaces ClickType), and `DataComponents.CUSTOM_DATA` (SkyBlock item ids).

## 2. Opening flow (fully automatic, as you chose)

```
Right-click holding a Jerry Box      → remember box type from the held item's custom_data id
"Open a Jerry Box" menu opens        → overlay appears the same frame; the menu is hidden and input to it is blocked
The box says "Click to open!"        → mod clicks that slot once (slot 22 in practice; found by content, not by number)
Wheel spins at full speed            → no target, no fake reward
Hypixel shows "It's rolling..."      → mod watches every slot of the menu
Result arrives                       → from a "Click to claim!" item (clicked once) or the "You claimed …" chat line
(Configured delay)                   → then the calculated slowdown lands exactly on the reward
Reveal (normal / rare / jackpot)     → overlay fades, HUD and held chat come back
```

Safety rails:
- The mod only clicks when the menu title is exactly `Open a Jerry Box`, the container id matches this session, and the
  slot holds a Jerry Box with "Click to open!" (or a "Click to claim!" result). At most one open click (+1 retry if
  Hypixel ignored it) and one claim click per session.
- An **Auto-click** setting (default ON) lets you turn automation off. With it off, you click the box yourself
  and the wheel still reacts.
- It never uses reward odds to pick anything; the odds only size the slices.

## 3. Files and classes

`src/main` holds pure Java with no client classes, so it can be unit-tested:

| Class | Purpose |
|---|---|
| `reward/JerryBoxType` | GREEN/BLUE/PURPLE/GOLDEN: SkyBlock id (`JERRY_BOX_*`), name, theme palette, reward pool |
| `reward/Reward` | record: key, display name, short label, weight, kind (COINS / SKILL_XP / ITEM), amount, skill, item id |
| `reward/RewardTables` | the four tables exactly as specified, in wheel order (§5) |
| `reward/RewardTier` | NORMAL / RARE / JACKPOT, derived from weight (§9) |
| `reward/RewardParser` | (name, lore, SkyBlock id) strings → exact `Reward`, or "unrecognised" ⏳ |
| `wheel/WheelLayout` | one slice per reward: equal sizes, except the 10M sliver (§5) |
| `wheel/SpinPlanner` | stopping math (§7), deterministic, no randomness in the result |

`src/client` holds everything Minecraft-client:

| Class | Purpose |
|---|---|
| `JerryRouletteClient` | wires everything up |
| `config/RouletteConfig`, `config/ConfigScreen` | JSON config + a simple vanilla-widget settings screen |
| `detect/JerryMenuTracker` | right-click tracking, menu detection, auto-clicks, reading the result |
| `anim/RouletteSession` | one box opening: type, container id, state, reward, timings |
| `anim/RouletteManager` | session queue, state machine, skip handling, timeout |
| `render/OverlayHost` | draws the overlay from both the HUD and screen hooks; blur guard |
| `render/WheelRenderer`, `render/WedgeRenderState` | the wheel (custom quad geometry) |
| `render/RevealRenderer`, `render/Effects` | reward card, rays, flash, confetti |
| `hide/SpoilerGuard` | hides HUD elements and holds chat while active (§10) |
| `icons/RewardIcons` | icon resolution: custom PNG → bundled icon → learned Hypixel item → vanilla fallback |
| `sound/RouletteSounds` | sound events plus one master volume |
| `debug/CaptureRecorder` | the recorder (kept for troubleshooting; manual-only in the release) |
| mixins | `ClientPacketListenerMixin`, `MultiPlayerGameModeMixin` (already done), `AbstractContainerScreenMixin` (hide the Jerry menu), `GuiRenderStateAccessor` (blur guard) |

## 4. Box detection

- **Type:** the held item's `minecraft:custom_data.id` (`JERRY_BOX_GREEN` / `_BLUE` / `_PURPLE` / `_GOLDEN`), taken at
  right-click via `UseItemCallback` (client side), with the main-hand item at menu-open as a fallback. It never uses
  the visible name alone. ⏳ Confirm Hypixel's custom_data layout and whether the menu shows the box type.
- **Trigger:** only the `Open a Jerry Box` menu starts a session, never a bare right-click.
- **Per-box toggles:** if a box type is disabled, the mod does nothing at all (no overlay, no clicks).

## 5. Reward tables and wheel order

Tables are copied verbatim from the spec; totals are 227 / 227 / 232 / 124. The weights are only used for the
"x% chance" line on the reveal and the rare/jackpot tier.

**Slice sizes (changed at your request):** every reward gets an **equal** slice, except **10,000,000 Coins**, which keeps
its weighted size (1/124 of the wheel = 2.9°) as a thin sliver. So Green and Blue have 10 × 36°, Purple has 11 × 32.7°,
and Golden has 9 × 39.7° plus the 2.9° 10M sliver. There is exactly one slice per unique reward.

Order around the wheel (valuable slices sit between common ones for natural near-misses):

- **Green / Blue:** Coins · Rune III · Farming · Candy · Jerry-chine · Foraging · 3D Glasses · Talisman · Mining · Stone
- **Purple:** as above, plus 300k Coins between Foraging and Talisman
- **Golden:** 500k · 10M (sliver) · Farming · 1M · Rune III · Foraging · Jerry-chine · 3D Glasses · Mining · Stone

## 6. Rendering

- **Where it draws:** a HUD element registered last (when no screen is open) plus `ScreenEvents.afterExtract` (when any
  screen is open, e.g. if you open your inventory mid-spin). While the overlay is active, the Hypixel Jerry menu
  itself is not drawn, so its animation can't be seen underneath.
- **Background:** a new layer, then blur (only if nothing else has blurred this frame, because vanilla crashes on a second
  blur, and only if your vanilla blur setting is above 0), then a dark translucent fill.
- **Style (changed at your request, following your slot-machine reference):** pixel art.
  - The machine: a dark cabinet with thick outlines, a coloured header plate ("GREEN / JERRY BOX" with gold stars), a
    lever that pulls down when the spin starts, a name plate ("JACKPOT!"-style) showing the slice under the pointer, and
    two white dots and a red button.
  - Around it: floating pixel icons (berries, bell, diamond, nugget) and twinkling "+" sparkles.
- **Wheel:** redrawn every frame on a grid of "art pixels". Each art pixel is a whole number of screen pixels and sits
  exactly on the screen grid, so it rotates as real chunky pixels and stays perfectly round at any resolution (tested at
  854×480, 1920×1080 and 2560×1440).
  - Layers: outline → rim in the theme colour with 20 chasing bulbs → slices (theme colours alternating, red for
    jackpots, gold rim band for rare/jackpot slices, flashing sliver for 10M) → dark dividers with cream pegs → hub
    with the Jerry Box item → red pointer with a flapper.
  - Every slice shows its item icon, drawn upright at 1 item pixel = 1 art pixel. The name plate shows the full name of
    whatever slice is passing under the pointer.
- **Themes:** Green, Blue, Purple and Gold palettes for the plate, rim, hub and slices. The cabinet, outlines and red
  accents are shared.
- **Scale and position:** config values; the art-pixel size snaps to whole screen pixels.

## 7. Animation and stopping math

States: `OPENING` (fade in, spin up 0.5 s) → `WAITING` (constant ω, about 540°/s) → `RESULT_DELAY` (configurable) →
`DECELERATING` → `SETTLE` (pointer flapper settles) → `REVEAL` → `CLOSING` → next queued session.
Timing uses `System.nanoTime` every frame, so motion is smooth rather than tied to game ticks.

**Landing:** the wheel angle θ increases clockwise and the pointer sits at the top. The slice under the pointer is at local
angle `(−θ) mod 360`. For target centre `c`:

```
θ_final ≡ −c (mod 360)
D       = θ_final − θ0 = 360·k + δ,   δ = (−c − θ0) mod 360
```

**Two-phase slowdown** gives a near-miss creep with continuous velocity and an exact finish:

- **Phase A** (fast slowdown): velocity `ω(u) = ω_h + (ω0 − ω_h)(1 − u)^m` from the current speed ω0 down to a crawl speed ω_h (about 20–30°/s),
  covering `D_A = D − d_h`. The exponent m is solved so the distance matches exactly, and k (extra full turns) is picked so
  the total time ≈ the configured duration.
- **Phase B** (creep): `ω(s) = ω_h(1 − s)^q` over the last `d_h` degrees, ending at **exactly 0 on the target centre**. d_h starts
  the creep inside the preceding slice or slices, so the wheel visibly almost stops, keeps crawling, ticks across a peg, and settles.

**Guarantees** (all unit-tested over thousands of random cases):
- final pointer = target centre (to 1e-6°)
- θ never decreases (no backwards motion)
- velocity is continuous at every phase boundary
- the result depends only on the reward and the current wheel state

Visual variety (creep length and crawl speed) never affects the landing.

**Honest near-misses:** the wheel never fully stops on a non-target slice, the reveal only starts after it stops on the
target, and the pointer ends mid-slice, so the result is unambiguous.

## 8. Skip, timeout, queue

- **Skip key** (default **Enter**; rebind in Controls → JerryRoulette). It works in-world and while a menu is open.
  If the reward is known, it jumps to the reveal. If not, it's remembered and the reveal happens as soon as Hypixel's
  result is read. It never invents a result.
- **Timeout:** if no reward is read within 30 s, the overlay closes with "Couldn't read the reward", held messages are
  released, and nothing is clicked.
- **Queue:** right-clicking another Jerry Box while a wheel is active is blocked client-side (no packet is sent). If a
  second Jerry menu opens anyway, it gets its own session tied to its own container id. It is hidden and processed in the
  background (open, read, close), and its wheel plays after the current one. Rewards can't be swapped between sessions.

## 9. Reveal and special rewards

The tier comes from the reward's weight within its box:

A cream "reel window" panel pops open over the wheel with a big icon, the reward name and "x% chance (1 in N)".

- **JACKPOT** (weight 1): 10M Coins, Jerry Rune III, Jerry 3D Glasses. Screen flash, pixel confetti, fast-chasing bulbs, a
  flashing gold/red "JACKPOT!" plate, a flashing name, and totem + firework sounds. Held about 4.5 s.
- **RARE** (weight 2–10): 1M Coins, 300k Coins, Jerry-chine Gun, Jerry Stone, Golden 100k XP. All bulbs blink, extra
  sparkles, a "BIG WIN!" plate and a challenge-complete fanfare. Held about 3.5 s.
- **NORMAL:** "YOU WON!" plate and a level-up chime. Held about 2.5 s.

## 10. Hiding the result (spoiler guard)

While a session is active:
- The HUD **chat, action bar, titles, scoreboard sidebar** (the purse would reveal coin amounts), **hotbar** and
  held-item name are hidden, and so are the health, food, armour, air and XP bars for a clean overlay.
- System chat messages are held and re-added in order after the reveal. The action bar is simply not drawn while active.
- The Jerry menu is hidden and its input blocked, apart from the skip key and Esc after a timeout.
- Everything returns to normal the moment the last session ends. Chat is never permanently changed.

## 11. Icons

**Your icons ship with the mod** (you supplied all ten): `assets/jerryroulette/textures/reward/<key>.png`, trimmed, upscaled
to 192×192 with sharpening, and marked for smooth filtering. Keys: `coins`, `farming_xp`, `foraging_xp`, `mining_xp`,
`jerry_chine_gun`, `jerry_stone`, `jerry_rune_iii`, `jerry_3d_glasses`, `jerry_candy`, `green_jerry_talisman`.

Resolution order per reward:
1. **Custom PNG**: `config/jerryroulette/icons/<reward_key>.png` (e.g. `jerry_rune_iii.png`), to override without rebuilding.
1b. The bundled icon above.
2. **Hypixel's own item**, learned from the Jerry menu when it shows it ⏳, cached in `config/jerryroulette/icon-cache.json`
   so it's available on every later spin. Minecraft loads the head textures itself, exactly as when you view the menu.
3. **Vanilla fallback**: gold nugget, ingot or block for the coin tiers; wheat, oak sapling or iron pickaxe for the XP types;
   sensible stand-ins for the items.

## 12. Sounds (vanilla for now, swappable)

The mod's own sound events are defined in `assets/jerryroulette/sounds.json`, each pointing at a vanilla sound. Replacing
one with a `.ogg` later needs no code change.

| Event | Default |
|---|---|
| wheel start | player attack sweep (whoosh) |
| peg tick | note block hat (pitch follows speed, rate-limited) |
| slice boundary click | wooden button click |
| slowdown begins | beacon deactivate |
| final pointer click | lever click |
| reveal | player level-up |
| rare reveal | challenge complete |
| jackpot reveal | totem + firework twinkle |

One **Sound Volume** setting scales all of them.

## 13. Settings

Stored in `config/jerryroulette.json`. Open with `/jerryroulette` (or an optional keybind, unbound by default).

- Enabled
- Auto-click
- Slowdown duration (default 6 s, range 2–15)
- Result delay (0.4 s, range 0–3)
- Sound volume
- Position X / Y
- Scale
- Green / Blue / Purple / Golden enabled
- Skip key (vanilla Controls)
- **Preview** buttons for each colour. These run a local demo spin on a reward you pick, labelled PREVIEW. They're for tuning
  scale, position and sound, and never touch Hypixel.

## 14. Testing

- **JUnit** (`./gradlew test`):
  - table totals and chances match the spec
  - angles sum to 360°
  - SpinPlanner exact landing, monotonic motion and continuous velocity over 10k random cases
  - RewardParser on the **real strings from your captures**: every coin amount (500k / 1M / 10M, etc.), every XP skill,
    every item, and rejection of unknown or ambiguous text
- **Client game tests** (`./gradlew runClientGameTest`): a real client, automated, with screenshots I inspect.
  - every theme; spinning, slowing, landed, and each reveal tier
  - skip with the result known and unknown
  - a two-box queue
  - scale and position
  - HUD hiding and chat restore
  - a **Hypixel simulator** in singleplayer that replays your captured slot-24 sequence into a fake `Open a Jerry Box`
    menu, covering the full pipeline including automatic clicks
- **Manual on Hypixel (you):** each box colour, reward shown equals reward received, auto-open and close, no spoilers
  visible, and fast re-opening (queue).

## 15a. Confirmed on Hypixel (capture 2026-09-21 21:06)

- The Jerry Box's custom_data is `{id:"JERRY_BOX_GREEN"}` at the top level, so right-click detection works.
- The menu `Open a Jerry Box` (9×6) has black panes, a **"Close"** barrier in slot 49, and the box in **slot 22** (not 24):
  "Green Jerry Box / Consume a jerry box to obtain a random reward! … / **Click to open!**".
- Clicking it plays a chest-open sound. Slot 22 then shows "Green Jerry Box / **It's rolling...**" while the panes around it
  flash and a note-block bass rises in pitch, for about 1.2 s.
- Then chat shows `§b ☺ §eYou claimed §620,000 coins §efrom the Jerry Box!` and the menu goes away. No "Click to claim!" item
  appeared that time.

**Changes made because of this:**
- The mod no longer uses slot numbers. It clicks whichever slot holds an exact "<Colour> Jerry Box" with "Click to open!".
- The result comes from **either** a "Click to claim!" item (then the mod clicks it once) **or** the claim chat line,
  whichever arrives first. Every claim line is also used as a cross-check.

## 15. Reward detection (from your screenshots + game log)

After Hypixel's animation, slot 24 holds a result item:

```
Green Jerry Box                 ← name: "<Green|Blue|Purple|Golden> Jerry Box"  → confirms the box type
You found coins!                ← kind line
                                   coins:  "Amount: 20,000"
                                   XP:     "Skill: Foraging" + "Amount: 2,500"      (from "You found Skill XP!")
                                   item:   "This is an uncommon drop!", "", "Jerry Candy", item lore…
                                           (from "You found a jerry item!")
Click to claim!                 ← final-state marker
```

**Parser rules** (exact, anchored matches on formatting-stripped lines; nothing loose):
- The result is only accepted when the name is exactly `<Colour> Jerry Box`, one `You found …!` line is present, **and**
  `Click to claim!` is present.
- **Coins:** `^Amount: ([\d,]+)$` is parsed to an integer and must **exactly equal** one of that box's coin rewards
  (500,000 vs 1,000,000 vs 10,000,000 can never be confused).
- **XP:** `^Skill: (Farming|Foraging|Mining)$` plus the amount, which must equal that box's XP amount.
- **Items:** the item-name line must equal one of the box's item names exactly (ignoring formatting and a leading symbol such
  as `◆`). Anything unrecognised is **not guessed**: the wheel fades out and shows Hypixel's own text instead.
- **Cross-check:** after the mod clicks "Click to claim!", Hypixel's chat line
  ` ☺ You claimed 2,500 Foraging XP from the Jerry Box!` is matched against the parsed reward. That message is also held
  back so it can't spoil the spin.

**Still unverified** (a single recorded opening confirms all of these; the design is safe without them):
1. What slot 24 shows **before** opening. Until confirmed, the auto-click requires: title matches, slot 24 not empty,
   and slot 24 not already a "Click to claim!" result.
2. What slot 24 shows **during** the animation. Frames are ignored unless they match the full result pattern above.
3. Whether claiming closes the menu. The session also completes if the menu stays open.
4. Exact names of the rarer items (Talisman, Jerry-chine Gun, Stone, Rune III, 3D Glasses) and the claim-message wording
   for coins and items. The parser logs any mismatch so it can be fixed from a single example.

## 16. History & stats (added)

Every recognised opening is appended to `config/jerryroulette/history.json` (box, reward key, timestamp; 5000 max).
Previews and unrecognised results are never recorded.

Open with `/jerryroulette stats` or the "History & stats" button in the settings screen:

- **Totals** for all boxes or one box: openings, coins, XP per skill, items.
- **Dry streaks**: boxes since the last rare and since the last jackpot.
- **Best drop** so far (rarest, then biggest).
- **Reward breakdown**: one row per reward with how often it actually came up versus Hypixel's listed chance
  ("seen / listed"). A single box lists all its rewards, including ones you've never had; the mixed view groups
  amounts ("Coins", "Farming XP") because the odds differ per box.
- **Recent** openings with relative times.
- **Clear history** (asks once for confirmation).

Aggregation lives in `jerry.modid.stats.RewardStats` and is unit-tested; the screen and the JSON file are client-side.
