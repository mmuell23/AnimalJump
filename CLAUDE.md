# MarlaGame – CLAUDE.md

Dieses Dokument beschreibt die Architektur und wichtige Konventionen des Spiels. Es dient als Orientierung für KI-Assistenten und Entwickler.

> **Wartungsregel:** Diese Datei soll immer aktualisiert werden, wenn sich größere Änderungen am Spiel ergeben (neue Level, neue Mechaniken, geänderte Konfigurationsobjekte, neue Spawn-Logik o. Ä.).

---

## Überblick

Browser-Seitenschnitt-Plattformer. Einzeldatei (`index.html`), Canvas 2D API, kein Build-Schritt, kein Framework.

- **Canvas:** 960 × 540 px
- **Game Loop:** `requestAnimationFrame` → `loop()` → `update()` + `draw()`
- **Spielzustände (`gs`):** `'select'` → `'select2'` → `'playing'` → `'levelend'` / `'gameover'` / `'win'`

---

## Wichtige Konstanten

```js
GROUND_Y    = H - 80           // 460 — Boden-Y (Füße der Spieler)
PLATFORM_Y  = GROUND_Y - 160   // 300 — Tier-1-Plattform
PLATFORM_Y2 = PLATFORM_Y - 100 // 200 — Tier-2-Plattform
PLATFORM_Y3 = PLATFORM_Y2 - 90 // 110 — Tier-3-Plattform
JUMP_VY     = -15              // normaler Sprungimpuls
MOVE_SPEED  = 3.5              // horizontale Laufgeschwindigkeit (px/frame)
LEVEL_FRAMES = 2400            // Länge eines Levels in Frames
BOX_W = 44, BOX_H = 46
```

---

## Konfigurationsobjekte

### `LEVEL_CFG[0..8]` — 9 Level

Pro Level: `speed`, `lionRatio`, `gapLo/Hi`, `ditchWMin/Max`, Sky-/Bodenfarben.

| Level | Speed | lionRatio | Besonderheit |
|---|---|---|---|
| 1 | 3.5 | 0.12 | Einführung, Hügel-Hintergrund |
| 2 | 4.5 | 0.22 | Wald-Hintergrund (2 Parallax-Schichten) |
| 3 | 5.5 | 0.35 | Sonnenuntergang |
| 4 | 6.5 | 0.48 | Roter Himmel |
| 5 | 7.0 | 0.50 | Etwas leichter; Graben-Rate halbiert; Sanduhr-Collectible |
| 6 | 7.0 | 0.62 | Gleiche Geschwindigkeit wie L5; Nacht-Stadtsilhouette; spezielle Kisten-Cluster |
| 7 | 7.0 | 0.72 | Gleiche Geschwindigkeit wie L6; Wüstenlandschaft; brennende Sonne; Sanddünen; Kakteen |
| 8 | 7.3 | 0.82 | Etwas schneller; Nordpol-Nacht; Aurora Borealis; Mondlicht; Schneedünen |
| 9 | 5.0 | 0.00 | Brückenlevel (`bridgeLevel: true`); ganzes Level ist Lava-Graben; kein Boden; nur Brücken |

> **Level 6 – Besonderheiten:**
> - Hintergrund: scrollende Hochhaus-Silhouetten mit Fenstern (`drawSkyscrapers()`), Mond statt Sonne.
> - Kasten-Cluster: max. 4 Spalten hoch; Stepping-Stone-Profil (Aufstieg → Peak → Abstieg), erzeugt via `chooseMaxHeight()` + `generateBoxBlock()`.
> - HUD zeigt `Level X / 8` (automatisch aus `LEVEL_CFG.length`).

> **Level 7 – Besonderheiten:**
> - Hintergrund: `drawDesert()` — Sanddünen (scrollend, Parallax 0.40), Kakteen-Silhouetten (`DESERT_CACTI`-Kachel), lodernde Sonne mit 6 Glow-Ringen.
> - Hitzeschlieren-Wolken (wie Level 4-Wolken: dunkel getönt).
> - Kasten-Cluster: gleiche Verteilung wie Level 6 (50 % → 4, 35 % → 3, 15 % → 2).

> **Level 8 – Besonderheiten:**
> - Hintergrund: `drawNorthPole()` — Sternenhimmel, Aurora Borealis (3 wellige Bänder in Grün/Cyan/Lila), Schneedünen, blasser Mond.
> - Boden: Eisblau-Weiß (`#D0E8F8`).
> - Kasten-Cluster: gleiche Verteilung wie Level 6+7.

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
                              invFrames, floatFrames, rocketFrames, jumpHeld, jumpBoost,
                              pendingDitchBonus, doubleJumped,
                              barkCooldown, scratchCooldown, wolfCooldown,
                              donkeyBoxCount, donkeyBoxCD }
```

- `players[]` — immer 1 oder 2 Einträge
- P1: Pfeiltasten; P2: W/S/A/D
- Linksgeschwindigkeit = `cfg.speed * 0.65` (immer sichtbar rückwärts, aber nie schneller als Scrolling)
- Variabler Sprung: Impuls −8, dann bis zu 14 Frames × 0.5 Boost = max vy −15
- Trampolin-Kiste: `JUMP_VY * 1.45` ≈ −21.75 (reicht bis zu den hohen Diamanten)
- Float-Modus (Trank): 450 Frames schwebend, Richtung per Up/Down-Taste

### Tier-Sonderfähigkeiten (↓-Taste)

Alle Sonderfähigkeiten sind **auch in der Luft** auslösbar (kein Bodencheck).

| Tier | Fähigkeit | Bedingung |
|---|---|---|
| Fuchs (0) | Doppelsprung | Nur in der Luft, einmal pro Sprung |
| Hund (1) | Einfrieren (WUFF) | `barkCooldown === 0` |
| Katze (2) | Wollknäuel schießen | `scratchCooldown === 0` |
| Esel (3) | Kisten-Bodenstampfer +25 | Nur beim Landen auf Kiste (`onGround`-Logik im Kollisionscode) |
| Wolf (4) | Schattenwolf rufen | `wolfCooldown === 0` |

---

##Kisten - Anordnung
Kisten können in unterschiedlichen Höhen auftauchen:
- eine Reihe
- zwei Reihen
- drei Reihen
- vier Reihen (ab Level 6)

Kisten stehen in Blöcken nebeneinander. Blöcke stehen entweder immer direkt nebeneinander ohne Lücken oder haben mindestens 2 Felder Platz zwischen den einzelnen Blöcken. Ab Level 6 sind es entweder gar keine Lücken zwischen Blöcken oder mindestens 5 Felder, damit genug Platz zum Landen ist.
Wenn Blöcke direkt nebeneinannderstehen, müssen mindestens 2 Blöcke nebeneinander die gleiche Höhe zum Aufstieg haben. 

Beispiel für eine Reihe (Kiste: x, keine Kiste: o):

xxooxxxoooxooxoooox

Beispiel für zwei Reihen:
ooxxoooooxoooooooxx
xxxxooooxxooooxooxx

Beispielfür drei Reihen:
ooooxxooooooooxxooo
ooxxxxooooxxxxxxooo
xxxxxxxxooxxooxxooo

Beispiel für vier Reihen:
ooooooxxooooooooooo
ooooooxxooooooooooo
ooooxxxxxxoooooxxoo
ooxxxxxxxxoooxxxxoo

Beispiel für Reihen, die so nicht erlaubt sind, weil die Lücken mit einer Kiste nicht betreten werden können:
oxxoxxx
xxxxxxx

Das hier hingegen geht:
oxxooxx
xxxxxxx

Das hier geht auch:
oxxooxx
xxxooxx

Wichtig ist:
- damit man auf 3 oder 4 Reihen springen kann, müssen davor Kisten stehen, von denen man abspringen kann. Diese sind maximal 2 Kisten hoch und mindestens 2 Kisten breit, damit die Fläche zum Absprung breit genug ist. 

## Kisten – Raketen-Punkte

Ab Level 1: Kiste per Rakete zerstören → **+5 Punkte**.
Ab Level 5: **+20 Punkte** (`level >= 5 ? 20 : 5` in `explodeBox()`).

---

## Spawn-Zustand (`spawner`)

Alle Spawn-Variablen in einem Objekt — wird in `initLevel()` atomar zurückgesetzt.

```js
spawner = {
  cd: 160,              // Spawn-Cooldown (frames)
  lastType: '',         // verhindert zwei Gegner hintereinander
  platformCount: 0,     // jede 2. Plattform bekommt Tier-1-Diamanten
  diamondBudget: 0,     // verbleibende Boden-/Kasten-Diamanten
  potionSpawned: false,
  rocketCount: 0,       // max 1 (Solo) / 2 (2P) pro Level
  highDiaCD: 0,         // Countdown bis zum nächsten Hochdiamant-Cluster
  clockSpawned: false,
  hourglassSpawned: false,
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

| Typ | Formel | L1 | L2 | L3 | L4 | L5 | L6 | L7 | L8 |
|---|---|---|---|---|---|---|---|---|---|
| Graben (ditch) | 22 % (L1–4), 11 % (L5+) | 22 % | 22 % | 22 % | 22 % | 11 % | 11 % | 11 % | 11 % |
| Löwe | `enemyTotal × (L3+: 0.38, L2: 0.57, L1: 1.00)` | 9 % | 10 % | 10 % | 14 % | 17 % | 21 % | 24 % | 28 % |
| Geier | `enemyTotal × (L3+: 0.28, L2: 0.43)` (L2+) | — | 7 % | 8 % | 10 % | 12 % | 15 % | 18 % | 20 % |
| Känguru | `enemyTotal × 0.34` (L3+) | — | — | 9 % | 13 % | 15 % | 19 % | 22 % | 25 % |
| Plattform | 52 % (2P) / 35 % (1P) des verbleibenden Anteils | | | | | | | | |
| Kasten-Cluster | Rest — alle Level via `spawnBoxCluster()` | | | | | | | | |

> `enemyTotal = lionRatio × (1 − ditchProb)`. Alle Prozentzahlen gerundet.

Nach einem Gegner: Mindestabstand 340 px (damit Stomp-Bounce sicher landet).

### Mindestabstand zwischen Kasten-Clustern

Position-basierter Check: `lastBoxRight > spawnX - minGapPx` (wobei `lastBoxRight` das Rechts-Ende des letzten Kasten-Hindernisses via `obstacles.reduce` ist).

- Level 1–5: `minGapPx = 2 * BOX_W`
- Level 6+: `minGapPx = 5 * BOX_W`

Ist der Abstand zu gering, wird statt eines Kasten-Clusters eine Plattform gespawnt.

---

## Kasten-Cluster-Funktionen

### `chooseMaxHeight()`
Gibt die maximale Spaltenhöhe (1–4) für den nächsten Cluster zurück, abhängig vom aktuellen Level:

| Level | Verteilung |
|---|---|
| 1–2 | 35 % → 2, 65 % → 1 |
| 3–4 | 35 % → 1, 35 % → 2, 30 % → 3 |
| 5 | 20 % → 1, 35 % → 2, 45 % → 3 |
| 6 | 15 % → 2, 35 % → 3, 50 % → 4 |

### `generateBoxBlock(maxH)`
Gibt ein Array von Spaltenhöhen zurück (Mountain-Profil: Aufstieg → Peak → Abstieg). **Kein interne Lücken** — alle Spalten direkt nebeneinander.

- `maxH === 1`: flacher Block, 1–5 Spalten, alle Höhe 1
- `maxH === 2`: Gruppen aus Höhe-2-Spalten, getrennt durch Lücken ≥2 Höhe-1-Spalten (1-Spalten-Lücken verboten — zu schmal zum Landen). Optional: führende/abschließende Höhe-1-Spalte.
- `maxH === 3`: `[1,1]` Anlauf, optional `[2,2]` (65 %), Peak mit 1–3 Spalten Höhe 3, optional Abstieg `[2,1]` oder `[2]`
- `maxH === 4`: `[1,1,2,2]` Anlauf (Pflicht), Peak 1–3 Spalten Höhe 4, optional Abstieg `[2,2,1]` oder `[2,2]`

> Die Anlauf-Spalten (max. 2 hoch, min. 2 breit) erfüllen die Regel, dass man auf hohe Türme abspringen kann.

### `spawnBoxCluster()`
Einheitliche Spawn-Funktion für alle Level:
1. Ruft `chooseMaxHeight()` und `generateBoxBlock(maxH)` auf
2. Setzt für jede Spalte `obstacles`-Einträge (Typ `'box'`, gestapelt von Boden bis Spaltenhöhe)
3. Platziert Diamanten, Trampolins und Collectibles wie bisher

---

## Hintergründe je Level

| Level | Hintergrund |
|---|---|
| 1 | Hügel (scrollende Halbkreise) |
| 2 | Wald — `drawForest()`: 2 Parallax-Schichten (`FOREST_BACK`, `FOREST_FRONT`) mit Bäumen (Stamm + Kronendach aus mehreren Kreisen) |
| 3–4 | Kein Natur-Element; Sonne, dunkle Wolken ab Level 4 |
| 5 | Wie 3–4 |
| 6 | Nacht: Mond, `drawSkyscrapers()` mit `CITY_BLDGS`-Kachel (960 px Periode, Parallax via `hillOff * 0.55`) |

---

> **Level 9 – Besonderheiten (Brückenlevel):**
> - Beim Level-Start: Mega-Graben `{ x: -W, w: W*200 }` + Startbrücke `w=310` → kein normaler Boden.
> - Neuer Obstacle-Typ `'bridge'`: One-Way-Landing wie Plattform; `y = GROUND_Y`; `h = 22`.
> - Spawn-Stream: `spawnNextBridge()` statt `spawnNext()` — spawnt nur Brücken, Geier, Kängurus.
> - Kängurus landen auf Brückenoberflächen und fallen in die Lava wenn sie die Kante verlassen.
> - Kängurus haben `bridgeObj`-Referenz + `jumpVxBias` für Kantenumkehr.
> - Hintergrund: `drawLavaChasm()` — Hitzeglühen, Glutpartikel, Cavernenvignette.
> - Brücken sind immun gegen Esel-Stampfer.
> - Keine Löwen; Gegner: Geier + Kängurus.

## Dev-Shortcuts (Tastatur)

Tasten **1–9** springen direkt zum entsprechenden Level (3 Leben, Punkte auf 0, aktuelle Tier-/Spieleranzahl-Einstellungen bleiben erhalten).

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
