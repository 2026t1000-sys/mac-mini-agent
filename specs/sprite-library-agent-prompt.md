# HoMM3-Style SVG Sprite Library — Agent Prompt Injection Reference

Use this as a reference/template when generating HTML game files that need quality sprites without external images.

---

## CORE SVG TEMPLATE STRUCTURE

Every quality unit sprite follows this pattern:

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="W" height="H" viewBox="0 0 W H">
  <defs>
    <!-- 1. Material gradient (primary body color with shading) -->
    <linearGradient id="steel" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#c8d8e8"/>
      <stop offset="40%" style="stop-color:#8aaabb"/>
      <stop offset="100%" style="stop-color:#445566"/>
    </linearGradient>
    <!-- 2. Highlight gradient (lighter, for lit surfaces) -->
    <linearGradient id="steelhi" x1="0%" y1="0%" x2="100%" y2="0%">
      <stop offset="0%" style="stop-color:#ddeeff;stop-opacity:0.9"/>
      <stop offset="100%" style="stop-color:#334455;stop-opacity:0.8"/>
    </linearGradient>
    <!-- 3. Skin gradient -->
    <radialGradient id="skin" cx="50%" cy="40%" r="60%">
      <stop offset="0%" style="stop-color:#f5c5a0"/>
      <stop offset="100%" style="stop-color:#c8855a"/>
    </radialGradient>
    <!-- 4. Glow filter (for magic/fire/eyes) -->
    <filter id="glow">
      <feGaussianBlur stdDeviation="2" result="blur"/>
      <feMerge><feMergeNode in="blur"/><feMergeNode in="SourceGraphic"/></feMerge>
    </filter>
  </defs>

  <!-- Draw order: shadow → feet → legs → torso → arms → weapon/shield → neck → head -->
  <ellipse cx="CX" cy="BOTTOM-4" rx="14" ry="4" fill="#000" opacity="0.35"/>
  <!-- ... rest of body ... -->
</svg>
```

---

## GRADIENT RECIPES BY MATERIAL

```
STEEL ARMOR:    #c8d8e8 → #8aaabb → #445566  (linearGradient, diagonal)
LEATHER:        #8B6914 → #6B4F10 → #3d2a05
MAGIC ROBE:     #3a1a6a → #220d4a → #120530
BONE:           #e8ddc8 → #c8b89a → #a89070
DRAGON SCALES:  #4a8a22 → #2a5a12 → #142d08
HORSE/BEAST:    #d4a855 → #a87c2a → #6a4a10
EAGLE FEATHER:  #8B6914 → #5a4008 → #2a1a00
FIRE:           #ffffff → #ffee44 → #ff6600 → #cc2200 (radial, center bright)
ICE:            #eeffff → #88ccee → #2266aa   (linear)
HOLY LIGHT:     #ffffff → #ffffaa → #ffcc44 → #ffaa00 (radial)
SHADOW/DARK:    #220d4a → #0d0520 → #000000
```

---

## FACTION COLOR PALETTES

```
Castle (blue/silver/red):   primary #c8d8e8, accent #cc2200, trim #ffd700
Rampart (green/wood/gold):  primary #4a8a22, accent #8B6914, trim #ffd700
Dungeon (purple/black/red): primary #3a1a6a, accent #cc2200, trim #8800cc
Necropolis (bone/green):    primary #e8ddc8, accent #00cc44, glow #44ff44
Tower (navy/silver/blue):   primary #0a1a4a, accent #8aaabb, trim #88aadd
Inferno (orange/dark red):  primary #882200, accent #ff6600, glow #ffcc00
Fortress (swamp/rust):      primary #3a5a2a, accent #6a4a2a, trim #8a5a2a
```

---

## 7 UNIT TYPE TEMPLATES (compressed — expand for actual use)

### KNIGHT (Castle Swordsman)
Key elements: steel helmet w/ visor slots + plume, breastplate w/ center ridge, pauldrons, sword (blade+crossguard+pommel), heater shield w/ emblem, greaves, boots
Gradients: steel, steelhi, skin, shield(#cc2200→#660000)
Distinctive: helmet plume (red path strokes), shield cross emblem, sword blade highlight

### ARCHER (Rampart)
Key elements: leather hood w/ feather, padded gambeson w/ lacing, bracer on bow wrist, longbow (thick curved path), quiver w/ arrow fletching showing
Gradients: leather, skin
Distinctive: bow is main identifying feature, quiver on back, determined expression

### WIZARD/MAGE (Tower)
Key elements: tall pointed hat w/ star, flowing robe w/ trim, long white beard, staff with glowing orb, spell sparks
Gradients: robe(#3a1a6a), orb(radial blue-white), skin
Filter: glow on orb, glowing eyes
Distinctive: hat + beard + staff orb = immediately recognizable

### SKELETON (Necropolis)
Key elements: actual anatomical bones (skull, ribcage w/ individual ribs, pelvis, femur, tibia, phalanges), glowing green eye sockets, rusty sword
Gradients: bone(#e8ddc8→#a89070)
Filter: green glow on eye circles
Distinctive: visible ribcage pattern is the key detail

### GRIFFIN (Castle/neutral)
Key elements: eagle head + beak + feather crest, eagle wings (brown, feather spar lines), lion haunches + tail w/ tuft, eagle front talons, lion back paws
Gradients: lionbody(#d4a855), eaglewing(#8B6914), eaglehead(#c8c080)
Distinctive: color split — dark brown eagle front, golden lion rear

### DRAGON (Dungeon/neutral)
Key elements: large body w/ scale texture (rows of curved strokes), dorsal spines, two spread wings w/ finger spars, 4 legs w/ claws, long neck, horned head, fire breath (radial gradient blob w/ glow filter)
Gradients: dscale(green), dbelly(gold), dwing(dark green), fireball(white→yellow→orange)
Filter: fire glow filter on breath
Distinctive: scale texture pattern + fire breath + wing span

### PEGASUS (Rampart/neutral)
Key elements: white horse body, large spread wings w/ individual feather lines + primary feather tips, magical mane/tail (purple-tinted), alicorn horn (golden), blue magical eyes
Gradients: whitecoat(white→#c0c0d8), winggrad(white→lavender)
Distinctive: pure white + purple mane + golden horn + massive wings

---

## ANIMATION CSS SNIPPETS

```css
/* Idle float (flying units) */
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-4px); }
}

/* Idle breathe (grounded units — apply to torso group) */
@keyframes breathe {
  0%, 100% { transform: scaleY(1); }
  50% { transform: scaleY(1.04) translateY(-1px); }
}

/* Undead eye glow */
@keyframes eyepulse {
  0%, 100% { filter: drop-shadow(0 0 2px #00ff00); }
  50% { filter: drop-shadow(0 0 8px #00ff00); }
}

/* Weapon glint (sparse, every 4-5s) */
@keyframes glint {
  0%, 80%, 100% { opacity: 0; }
  85% { opacity: 0.9; }
  90% { opacity: 0.2; }
}

/* Attack swing (trigger via JS class toggle) */
@keyframes swordswing {
  0% { transform: rotate(0deg); }
  20% { transform: rotate(-45deg); }
  60% { transform: rotate(70deg); }
  80% { transform: rotate(15deg); }
  100% { transform: rotate(0deg); }
}
.sword-arm.attacking {
  animation: swordswing 0.5s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
}

/* Death fall */
@keyframes deathfall {
  0% { transform: rotate(0deg) translateY(0); opacity: 1; }
  30% { transform: rotate(-15deg) translateY(-5px); opacity: 1; }
  100% { transform: rotate(-90deg) translateY(35px); opacity: 0; }
}
.unit.dying { animation: deathfall 0.8s ease-in forwards; }
```

---

## SVG SMIL ANIMATION SNIPPETS

```svg
<!-- Continuous rotation (magic circles, spin effects) -->
<animateTransform attributeName="transform" type="rotate"
  from="0 CX CY" to="360 CX CY" dur="4s" repeatCount="indefinite"/>

<!-- Pulsing (orbs, hearts, gems) -->
<animate attributeName="r" values="8;12;8" dur="1.5s" repeatCount="indefinite"/>
<animate attributeName="opacity" values="0.7;1;0.7" dur="1.5s" repeatCount="indefinite"/>

<!-- Orbiting particle -->
<circle r="4" fill="#88aaff">
  <animateMotion dur="2s" repeatCount="indefinite"
    path="M0,0 m-25,0 a25,25 0 1,1 50,0 a25,25 0 1,1 -50,0"/>
</circle>

<!-- Color flicker (fire/torch) -->
<animate attributeName="fill"
  values="#ff6600;#ff8800;#ffcc00;#ff6600;#ff4400;#ff6600"
  dur="0.3s" repeatCount="indefinite"/>
```

---

## SPELL EFFECTS PATTERNS

```
LIGHTNING:   jagged polyline × 3 (outer glow, mid, white core) + branch polylines + impact circle
FIREBALL:    irregular flame path (lobed polygon) + radialGradient (white→yellow→orange→transparent) + ember circles + filter glow
HOLY LIGHT:  cross rects + radialGradient + rotating circle ring + star polygons + sparkle paths
ICE/BLIZZARD: snowflake (6-armed line group with rotate transforms) + shard polygons + ice gradient
POISON:      green radialGradient blob + feTurbulence filter for bubbling texture
DARK MAGIC:  purple radialGradient + rotating rune circle (dashed stroke) + feDisplacementMap for warp

FIRE GLOW FILTER:
<filter id="fireglow">
  <feGaussianBlur stdDeviation="3" result="blur"/>
  <feColorMatrix in="blur" type="matrix"
    values="1 0 0 0 0.3  0 0.4 0 0 0  0 0 0 0 0  0 0 0 0.9 0" result="glow"/>
  <feMerge><feMergeNode in="glow"/><feMergeNode in="SourceGraphic"/></feMerge>
</filter>

MAGIC BLUE GLOW FILTER:
<filter id="magiccglow">
  <feGaussianBlur stdDeviation="2" result="blur"/>
  <feColorMatrix in="blur" type="matrix"
    values="0 0 0 0 0.3  0 0 0 0 0.6  0 0 0 0 1  0 0 0 0.8 0" result="glow"/>
  <feMerge><feMergeNode in="glow"/><feMergeNode in="SourceGraphic"/></feMerge>
</filter>
```

---

## TERRAIN TILE PATTERNS

```
GRASS:  linearGradient(#4aaa2a→#2a6a14) + pattern{grass blade paths}
STONE:  linearGradient(#888888→#555555) + pattern{offset masonry rects + mortar lines}
WATER:  linearGradient(#1a6aaa→#0a2a55) + pattern{wave crests as curved paths + specular ellipses}
SAND:   radialGradient(#d4aa55→#a87a2a) + pattern{small ellipse pebbles}
LAVA:   linearGradient(#882200→#440000) + pattern{orange crack paths + lavaspot radialGradient hotspots}
SNOW:   white + pattern{subtle blue shadows + snowflake dots}
DIRT:   #6a4a2a + pattern{pebble ellipses + variation patches}

All terrain tiles: 64x64 or 80x80 viewBox, use <pattern> for seamless tiling
Apply to hex/square cells: <polygon ... fill="url(#terrainPattern)"/>
```

---

## QUICK QUALITY CHECKLIST

Before finalizing any sprite:
- [ ] Shadow ellipse at feet (black, opacity 0.3-0.4)
- [ ] Minimum 2 gradients (body + highlight)
- [ ] Eyes/face detail (even on creatures)
- [ ] Weapon or distinctive prop
- [ ] 30+ SVG elements for humanoids, 50+ for creatures
- [ ] Faction-correct color palette
- [ ] Glow filter on any fire/magic/glowing elements
- [ ] Texture lines for fur/scales/armor ridges
- [ ] White highlight ellipse on metal surfaces (opacity 0.1-0.15)
- [ ] Readable silhouette at thumbnail size

---

## FILES IN THIS LIBRARY

- `sprite-library-part1.html` — Knight, Archer, Wizard SVG code (rendered + source)
- `sprite-library-part2.html` — Dragon, Skeleton, Pegasus + Animation systems
- `sprite-library-part3.html` — Griffin, Spell effects, Terrain tiles, Quality guide
- `sprite-library-agent-prompt.md` — THIS FILE: condensed reference for AI prompts
