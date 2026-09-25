# SkyCasino

A client-side Fabric mod for Minecraft **26.2** that turns two Hypixel SkyBlock
moments into animations:

- **Archfiend Dice** rolls become a pixel-art **slot machine**
- **Jerry Box** openings become a spinning **prize wheel**

Both are **purely visual**. Hypixel decides every outcome; the mod reads the
result and animates toward it. It never predicts, alters, rerolls or invents a
result — and if no result arrives, no reward is ever shown.

Either half can be switched off in settings.


# Discord: https://discord.gg/dJv5Ryn8zU
---

## Download

**A compiled jar is available — you do not need to build anything.** Grab the
latest `skycasino-<version>.jar` from the
[Releases](../../releases) page, or from `build/libs/` if you have cloned the
repository.

### Installing

1. Install [Fabric Loader](https://fabricmc.net/use/installer) 0.19.5+ for
   Minecraft 26.2.
2. Drop **`skycasino-1.1.0.jar`** and
   [Fabric API](https://modrinth.com/mod/fabric-api) (`0.161.0+26.2` or later)
   into your `mods` folder.
3. Launch the game and join Hypixel.

## Requirements

| | |
|---|---|
| Minecraft | 26.2 |
| Fabric Loader | 0.19.5+ |
| Fabric API | 0.161.0+26.2 |
| Other mods | None. SkyHanni, NEU and SBA are **not** needed. |

Client-side only — it does nothing on a server and nothing outside Hypixel
SkyBlock.

---

## Features

### The slot machine (Archfiend Dice)

Right-click an Archfiend Dice and the cabinet drops in and starts spinning. The
reels **cannot** stop until Hypixel's chat message arrives, because the landing
symbol is derived from it.

Both dice roll **1–7**:

| Face | Archfiend Dice | High Class Archfiend Dice |
|---|---|---|
| 1–5 | health (−120 / −60 / −30 / +30 / +60) | health (−300 / −200 / −100 / +100 / +200) |
| 6 | health and **15,000,000 coins** | +300 health and **100,000,000 coins** |
| 7 | **Archfiend Dye** | **Archfiend Dye** |

- **Health rolls** show the figure read straight from the message. Each face has
  its own weight: the worst lands in a washed-out red with a hard shake, the best
  arrives bright green with sparkles.
- **A 6** gets the coin sequence — gold flash, shockwaves, a rotating sunburst, a
  coin fountain and a counter running up to that dice's payout.
- **A 7** is the rarest result and gets the longest sequence of all, built around
  the Archfiend Dye itself in crimson and gold rather than coins.
- **Near-miss tease.** On roughly two thirds of non-jackpot rolls the final reel
  crawls past the money bag before settling on what you actually got.

Health figures are never hardcoded, so a retune on Hypixel's side still shows the
right number. Four cabinet themes: Purple, Teal, Midnight and Emerald, each with
gold trim.

### The prize wheel (Jerry Boxes)

Right-click a Jerry Box and the wheel takes over. The Jerry menu is hidden and
its input blocked, the wheel spins with no target until Hypixel reveals the
reward, then lands exactly on it. All four tiers are supported — Green, Blue,
Purple and Golden — with slice sizes taken from the real reward tables.

Reward icons resolve from your own PNG, then the bundled icon, then the item
Hypixel actually showed.

### Shared

- **Spoilers stay hidden.** While either animation plays, chat, the action bar,
  titles, the hotbar and the scoreboard sidebar (purse included) are not drawn,
  and incoming messages are held back and released in order afterwards.
- **Statistics** for both halves — see below.
- **Skip key** to cut any animation short. It can never change the result: if the
  result is not known yet, the overlay simply closes showing nothing.

---

## Commands

| Command | Does |
|---|---|
| `/skycasino` | Open settings |
| `/skycasino stats` | Open statistics |
| `/skycasino skip` | Skip whatever is animating |
| `/skycasino status` | What the mod currently thinks is happening |
| `/skycasino dice` | Dice settings |
| `/skycasino dice simulate <1-7>` | Play a dice animation locally |
| `/skycasino dice debug <true\|false>` | Log every incoming chat message |
| `/skycasino dice testmode <true\|false>` | Allow the mod to run outside SkyBlock |
| `/skycasino jerry preview <box> [n]` | Play a wheel locally |
| `/skycasino jerry capture start\|stop` | Troubleshooting recorder |
| `/diceroll [1-7]` | Shorthand dice test roll |
| `/diceroll highclass <1-7>` | Test roll on the High Class dice |

Previews and test rolls are local demos. Nothing is sent to any server, and they
are deliberately **excluded from your statistics**.

## Keybinds

| Action | Default |
|---|---|
| Skip the animation | `X` |
| Open settings | unbound — use `/skycasino` |

Rebindable from the settings screen or the vanilla Controls screen.

## Settings

One window, three tabs:

- **General** — turn either half on or off, the auto-click toggle, and the skip
  key.
- **Dice** — animation style and duration, cabinet theme, position, scale,
  volume, result delay, timeout, per-dice toggles, and the `/pc` announcement.
- **Jerry Box** — slowdown, result delay, volume, position, scale, which box
  tiers are enabled, capture logging and per-box previews.

Stored in `config/skycasino/`. Files from the two mods this was merged from are
carried over automatically the first time it runs.

### Party announcements

Optional and **off by default**. When enabled, a coin jackpot or an Archfiend Dye
posts a customisable message via `/pc`, with separate templates per dice and a
live preview of what would be sent. Rate-limited to one message a minute.

This sends a real message from your account. Automated chat can read as spam to
other players, so it stays off unless you turn it on.

## Statistics

`/skycasino stats`, one tab per half.

- **Dice** — a bar per face using the reels' own symbols, total rolls, jackpot
  and dye counts with observed rates (`1 in 47`), longest drought, and coins won
  against coins spent.
- **Jerry Box** — boxes opened per tier, coins and skill XP totals, a full reward
  breakdown with observed against listed chances, dry streaks and recent
  openings.

---

## ⚠️ Automation note

**The Jerry half clicks the box and the claim slot for you, and this is on by
default.** That is GUI automation on Hypixel and carries real account risk.

Turn **Jerry auto-click** off on the General settings tab if you would rather
click yourself — the wheel still reacts either way. The dice half never
automates anything; it only watches your own right-click and reads chat.

---

## Building from source

Minecraft 26.2 needs **Java 25**, and Fabric Loom requires Gradle itself to run
on it. The build declares a Java 25 toolchain and `settings.gradle` includes the
foojay resolver, but the Gradle daemon still has to start on a JDK 25:

```bash
JAVA_HOME=/path/to/jdk-25 ./gradlew build
```

| Task | Does |
|---|---|
| `./gradlew build` | Builds `build/libs/skycasino-1.1.0.jar` |
| `./gradlew test` | JUnit tests for the pure logic in `src/main` |
| `./gradlew runClient` | Launches a dev client |
| `./gradlew runClientGameTest` | Automated client checks, with screenshots |

## Project structure

| Package | Holds |
|---|---|
| `skycasino.core` | Entrypoint, keybinds, command root, overlay host, spoiler guard, shared screens |
| `skycasino.dice` | Everything for the slot machine |
| `skycasino.jerry` | Everything for the prize wheel |
| `skycasino.mixin` | The few mixins, shared |

`src/main` is pure Java with no client classes so it can be unit-tested;
`src/client` holds everything Minecraft-side.

Two implementation details worth knowing if you are reading the code:

- **One blur per frame.** Minecraft throws if the blur post-process is requested
  twice in a frame. `OverlayHost` is the only place that asks, which is why both
  animations are hosted there rather than registering their own HUD elements.
- **Nothing is textured.** The slot machine, its symbols and the Archfiend Dye
  are drawn in code on a chunky pixel grid, so they stay sharp at any scale.

## Known limitations

- The wording of the dice **6** and **7** messages, and of every **High Class**
  message, has not been confirmed against a real roll. If one differs, that roll
  is not picked up — and a warning with the real text is written to `latest.log`
  so the pattern can be corrected.
- Sound effects are fitting vanilla sounds rather than custom audio. They are
  registered under the mod's own ids, so a resource pack can replace them.

## License

MIT — see [LICENSE](LICENSE).
