# MarlaGame – CLAUDE.md

Dieses Dokument beschreibt die Architektur und wichtige Konventionen des Spiels. Es dient als Orientierung für KI-Assistenten und Entwickler.

## Überblick

Browser-Seitenschnitt-Plattformer. Einzeldatei (`index.html`), Canvas 2D API, kein Build-Schritt, kein Framework.

- **Canvas:** 960 × 540 px
- **Game Loop:** `requestAnimationFrame` → `loop()` → `update()` + `draw()`
- **Spielzustände (`gs`):** `'select'` → `'select2'` → `'playing'` → `'levelend'` / `'gameover'` / `'win'`

---

## Wichtige Konstanten

```js
GROUND_Y    = H - 80          // 460 — Boden-Y (Füße der Spieler)
PLATFORM_Y  = GROUND_Y - 160  // 300 — Tier-1-Plattform
PLATFORM_Y2 = PLATFORM_Y - 100 // 200 — Tier-2-Plattform
PLATFORM_Y3 = PLATFORM_Y2 - 90 // 110 — Tier-3-Plattform
JUMP_VY     = -15             // normaler Sprungimpuls
MOVE_SPEED  = 3.5             // horizontale Laufgeschwindigkeit (px/frame)
LEVEL_FRAMES = 2400           // Länge eines Levels in Frames
BOX_W = 44, BOX_H = 46
```

---

## Konfigurationsobjekte

### `LEVEL_CFG[0..4]`
Pro Level: `speed`, `lionRatio`, `gapLo/Hi`, `ditchWMin/Max`, Sky-/Bodenfarben.

### `DIAMOND`
Zentrales Registry für alle Diamant-Typen. **Hier anpassen, nicht an einzelnen Spawn-Stellen.**

```js
DIAMOND = {
  box:   { color: '#38c8ff', pts: 10  },  // Boden & Kästen
  tier1: { color: '#38c8ff', pts: 10  },  // Tier-1-Plattform
  tier2: { color: '#ff2244', pts: 30  },  // Tier-2-Plattform
  tier3: { color: '#cc44ff', pts: 50  },  // Tier-3-Plattform
  high:  { color: '#FFD700', pts: 20  },  // schwebend (nur per Trank erreichbar)
}
```

> **Hinweis:** `high.pts` ist bewusst auf **20** gesetzt (manuell angepasst). Die Render-Logik erkennt hohe Diamanten via `d.pts >= DIAMOND.high.pts` — wenn dieser Wert geändert wird, zieht die Erkennung automatisch mit.

### `PLATFORM_TIERS[0..1]`
Treibt die Schleife in `spawnPlatform()`. Für eine vierte Ebene einfach einen Eintrag anhängen.

```js
PLATFORM_TIERS = [
  { y: PLATFORM_Y2, minN: 1, maxN: 3, spawnChance: 0.45, diaChance: 0.5,  diamond: 'tier2' },
  { y: PLATFORM_Y3, minN: 1, maxN: 2, spawnChance: 0.60, diaChance: 1.0,  diamond: 'tier3' },
]
```

---

## Spieler

```js
makePlayer(x, animalIdx) → { x, y, vy, vx, onGround, runT, animalIdx,
                              invFrames, floatFrames, jumpHeld, jumpBoost,
                              pendingDitchBonus }
```

- `players[]` — immer 1 oder 2 Einträge
- P1: Pfeiltasten; P2: W/S/A/D
- Linksgeschwindigkeit = `cfg.speed * 0.65` (immer sichtbar rückwärts, aber nie schneller als Scrolling)
- Variabler Sprung: Impuls −8, dann bis zu 14 Frames × 0.5 Boost = max vy −15
- Trampolin-Kiste: `JUMP_VY * 1.45` ≈ −21.75 (reicht bis zu den hohen Diamanten)
- Float-Modus (Trank): 450 Frames schwebend, Richtung per Up/Down-Taste

---

## Spawn-Zustand (`spawner`)

Alle Spawn-Variablen in einem Objekt — wird in `initLevel()` atomar zurückgesetzt.

```js
spawner = {
  cd: 160,            // Spawn-Cooldown (frames)
  lastType: '',       // verhindert zwei Gegner hintereinander
  platformCount: 0,   // jede 2. Plattform bekommt Tier-1-Diamanten
  diamondBudget: 0,   // verbleibende Boden-/Kasten-Diamanten
  potionSpawned: false,
  highDiaCD: 0,       // Countdown bis zum nächsten Hochdiamant-Cluster
}
```

---

## `update()` — Aufteilung

```
update()
  ├─ updatePlayer(p, cfg)    — Physik, Sprung, Plattform-Landing pro Spieler
  ├─ updateWorld(cfg)        — Clouds, Timer, Popups, Obstacles/Ditches bewegen & cullen
  ├─ updateCollectibles(cfg) — Herzen, Tränke, Diamanten einsammeln
  ├─ updateEnemies()         — Geier-Oszillation
  ├─ updateSpawner(cfg)      — Spawn-Stream, Hochdiamanten, Flagge
  └─ updateCollisions()      — AABB-Kollision Spieler ↔ Gegner/Kästen
```

---

## Spawn-Stream (`spawnNext`)

Wahrscheinlichkeiten je Frame:

| Typ | Bedingung |
|---|---|
| Graben (ditch) | 22 % |
| Löwe | `lionRatio * 0.6`, nicht nach Gegner |
| Geier | `lionRatio * 0.4`, nicht nach Gegner |
| Plattform | 52 % (2P) / 35 % (1P) des verbleibenden Anteils |
| Kasten-Cluster | Rest |

Nach einem Gegner: Mindestabstand 340 px (damit Stomp-Bounce sicher landet).

---

## Kollision

- AABB mit Margin 11 px (`overlaps()`)
- Einweg-Plattform-Landing: `p.vy >= 0 && prevY <= o.y + 6`
- Trampolin-Landing löst Super-Sprung aus statt `onGround = true`

---

## Sound

Web Audio API, lazy `AudioContext`. Alle Sounds prozedural:
`playJump`, `playStep`, `playPunch`, `playPling`, `playHeart`, `playPotion`, `playPuff`, `playBoing`

---

## Tier-System (Plattformen)

`spawnPlatform(n)`:
1. 2 Trittstein-Kästen auf Boden
2. n-breite Tier-1-Plattform bei `PLATFORM_Y`
3. Loop über `PLATFORM_TIERS` — bricht ab, wenn `Math.random() >= tier.spawnChance`

---

## Was NICHT hier steht

Farben, Sprite-Koordinaten, Level-Balancing-Werte — alles direkt im Code bzw. in den Konfigurationsobjekten oben nachschlagen.
