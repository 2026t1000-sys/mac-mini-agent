# Web Audio API Game Sound Synthesis Library

Complete research and implementation guide for synthesizing high-quality game sounds
using only the Web Audio API — no external audio files, no CDN libraries.

---

## 1. Web Audio API Fundamentals

### Core Concepts

The Web Audio API works as a **signal routing graph** of `AudioNode` objects connected
together. Everything flows from a source node → through processing nodes → to the
`AudioContext.destination` (speakers).

**Key node types:**

| Node | Purpose |
|------|---------|
| `OscillatorNode` | Generates periodic waveforms (sine, square, sawtooth, triangle) |
| `AudioBufferSourceNode` | Plays raw PCM data (white noise, custom waveforms) |
| `GainNode` | Controls amplitude/volume |
| `BiquadFilterNode` | Lowpass, highpass, bandpass, notch filters |
| `DynamicsCompressorNode` | Prevents clipping, adds "punch" |
| `ConvolverNode` | Convolution reverb |
| `DelayNode` | Echo/delay effects |
| `StereoPannerNode` | Left/right panning |

### Parameter Automation (The Key to Good Sounds)

The `AudioParam` interface lets you schedule value changes with sample-accurate timing:

```javascript
const now = ctx.currentTime;
gain.gain.setValueAtTime(1.0, now);                        // instant set
gain.gain.linearRampToValueAtTime(0, now + 0.5);          // linear fade
gain.gain.exponentialRampToValueAtTime(0.001, now + 0.5); // exponential (sounds more natural)
gain.gain.setTargetAtTime(0, now + 0.1, 0.05);            // RC-style decay (very natural)
```

**Rule:** Use `exponentialRampToValueAtTime` for amplitude envelopes — exponential decay
sounds more natural to humans. Use `linearRamp` only for pitch glides.
`exponentialRamp` cannot go to exactly 0 (use 0.001 instead).

### AudioContext Lifecycle

```javascript
// Create context (always starts suspended on Safari/iOS)
const ctx = new (window.AudioContext || window.webkitAudioContext)();

// Must resume after a user gesture (click, tap, keypress)
document.addEventListener('click', () => {
    if (ctx.state === 'suspended') ctx.resume();
}, { once: false });
```

### Cross-Browser / Safari / iOS Workarounds

Safari has historically lagged behind Chrome on Web Audio support. Key mitigations:

1. **Always use the vendor prefix fallback:**
   ```javascript
   const AudioContext = window.AudioContext || window.webkitAudioContext;
   ```

2. **AudioContext must start after user gesture.** Gate all sound behind an interaction.
   The canonical pattern — attach to the first meaningful user interaction:
   ```javascript
   function ensureAudioCtx() {
       if (!window._audioCtx) {
           window._audioCtx = new (window.AudioContext || window.webkitAudioContext)();
       }
       if (window._audioCtx.state === 'suspended') {
           window._audioCtx.resume();
       }
       return window._audioCtx;
   }
   // Attach to game canvas or document click
   document.addEventListener('click', ensureAudioCtx, { once: true });
   ```

3. **Safari iOS volume:** Do not use `HTMLMediaElement.volume`. Use Web Audio GainNodes.

4. **Context interruption:** When the user backgrounds the app, the context may become
   `interrupted` (iOS). Handle via the `statechange` event:
   ```javascript
   ctx.addEventListener('statechange', () => {
       if (ctx.state === 'interrupted') ctx.resume();
   });
   ```

5. **Node reuse:** `OscillatorNode` and `AudioBufferSourceNode` are **one-shot** — once
   stopped they cannot be restarted. Always create new nodes for each sound.

---

## 2. What Makes Sounds Good vs Bad

### The "Horrible Beeping" Problem

Basic implementations that only do:
```javascript
osc.type = 'sine'; osc.frequency.value = 440; osc.start(); osc.stop(ctx.currentTime + 0.1);
```
sound terrible because:
- **No amplitude envelope** — instant on/off creates clicks (DC offset artifacts)
- **Pure sine wave** — no harmonic content, sounds like a test tone
- **No pitch movement** — real objects change pitch during impact
- **No noise component** — most "impact" sounds have broadband noise content
- **No filtering** — all frequencies equally present, unnatural

### Parameters That Most Affect Quality

1. **Envelope shape (ADSR)** — The most important. Attack time, decay curve, sustain level,
   release curve. Percussive sounds have near-zero attack, fast exponential decay.

2. **Noise layer** — Mixing white/pink noise with oscillators adds realism to impacts, breath,
   friction. A pure oscillator sounds electronic; oscillator + noise sounds physical.

3. **Frequency modulation / pitch sweep** — Real impacts have a pitch that drops rapidly.
   Frequency sweep from high→low (exponential) on a kick drum is the entire trick.

4. **Filter sweep** — Shape the frequency content over time. A lowpass filter closing down
   during decay makes sounds "thud" naturally.

5. **Distortion / waveshaping** — Adds harmonics to make sounds "fat". Crucial for making
   bass sounds and impacts feel heavy.

6. **DynamicsCompressor on master** — Glues all sounds together, prevents clipping, adds
   perceived loudness and "punch". Always put one at the end of the chain.

### Acceptable vs Great

| Aspect | Acceptable | Great |
|--------|------------|-------|
| Envelope | Linear ramp to 0 | Exponential decay with proper attack |
| Waveform | Sine only | Layered oscillators + noise |
| Pitch | Static | Pitch sweep/glide during sound |
| Filtering | None | Dynamic filter envelope |
| Polish | Raw signal | DynamicsCompressor + slight reverb |
| Variation | Same every time | Random ±5-10% pitch/timing variation |

---

## 3. The Complete Sound Library

Embed this entire block in game generation prompts. It is self-contained and requires
no external files or dependencies.

```javascript
// ============================================================
// GAME AUDIO SYSTEM — Web Audio API synthesis, no external files
// ============================================================

const GameAudio = (() => {
    let _ctx = null;
    let _masterGain = null;
    let _compressor = null;
    let _musicNodes = [];
    let _musicGain = null;
    let _sfxGain = null;

    // --------------------------------------------------------
    // INITIALIZATION
    // --------------------------------------------------------

    function init() {
        if (_ctx) return _ctx;
        const AC = window.AudioContext || window.webkitAudioContext;
        if (!AC) return null;
        _ctx = new AC();

        // Master chain: sfx/music → master gain → compressor → destination
        _compressor = _ctx.createDynamicsCompressor();
        _compressor.threshold.value = -18;
        _compressor.knee.value = 6;
        _compressor.ratio.value = 4;
        _compressor.attack.value = 0.003;
        _compressor.release.value = 0.15;
        _compressor.connect(_ctx.destination);

        _masterGain = _ctx.createGain();
        _masterGain.gain.value = 1.0;
        _masterGain.connect(_compressor);

        _sfxGain = _ctx.createGain();
        _sfxGain.gain.value = 1.0;
        _sfxGain.connect(_masterGain);

        _musicGain = _ctx.createGain();
        _musicGain.gain.value = 0.4;
        _musicGain.connect(_masterGain);

        // iOS/Safari autoplay: context starts suspended, resume on user gesture
        const resume = () => { if (_ctx.state === 'suspended') _ctx.resume(); };
        document.addEventListener('click', resume);
        document.addEventListener('touchstart', resume);
        document.addEventListener('keydown', resume);
        _ctx.addEventListener('statechange', () => {
            if (_ctx.state === 'interrupted') _ctx.resume();
        });

        return _ctx;
    }

    function getCtx() {
        return _ctx || init();
    }

    // --------------------------------------------------------
    // UTILITIES
    // --------------------------------------------------------

    // Create white noise buffer (1-2 seconds, loopable)
    function createNoiseBuffer(ctx, duration) {
        const length = ctx.sampleRate * (duration || 1);
        const buffer = ctx.createBuffer(1, length, ctx.sampleRate);
        const data = buffer.getChannelData(0);
        for (let i = 0; i < length; i++) {
            data[i] = Math.random() * 2 - 1;
        }
        return buffer;
    }

    // Generate a synthetic reverb impulse response (no file needed)
    // Based on exponentially decaying noise — sounds like a small stone room
    function createReverbBuffer(ctx, duration, decay) {
        duration = duration || 1.5;
        decay = decay || 2.0;
        const rate = ctx.sampleRate;
        const length = rate * duration;
        const impulse = ctx.createBuffer(2, length, rate);
        for (let ch = 0; ch < 2; ch++) {
            const data = impulse.getChannelData(ch);
            for (let i = 0; i < length; i++) {
                data[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / length, decay);
            }
        }
        return impulse;
    }

    // Lazy-init reverb node (expensive to create, reuse it)
    let _reverb = null;
    function getReverb(ctx) {
        if (!_reverb) {
            _reverb = ctx.createConvolver();
            _reverb.buffer = createReverbBuffer(ctx);
            _reverb.connect(_sfxGain);
        }
        return _reverb;
    }

    // Random variation: returns value ± percent
    function vary(value, percent) {
        return value * (1 + (Math.random() * 2 - 1) * percent);
    }

    // --------------------------------------------------------
    // VOLUME CONTROL
    // --------------------------------------------------------

    function setMasterVolume(v) { if (_masterGain) _masterGain.gain.value = v; }
    function setMusicVolume(v)  { if (_musicGain)  _musicGain.gain.value  = v; }
    function setSfxVolume(v)    { if (_sfxGain)    _sfxGain.gain.value    = v; }

    function fadeMusic(targetVol, duration) {
        if (!_musicGain) return;
        const ctx = getCtx();
        _musicGain.gain.cancelScheduledValues(ctx.currentTime);
        _musicGain.gain.setValueAtTime(_musicGain.gain.value, ctx.currentTime);
        _musicGain.gain.linearRampToValueAtTime(targetVol, ctx.currentTime + duration);
    }

    // --------------------------------------------------------
    // SFX: HIT / IMPACT (sword hitting shield — metallic thwack)
    // --------------------------------------------------------
    // Technique: layered transient oscillator + noise burst + bandpass filter
    // The pitch sweep down + metallic ring make it feel like metal-on-metal

    function playHitSound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        // Layer 1: transient tone with rapid pitch drop (the "thud" body)
        const osc = ctx.createOscillator();
        const oscGain = ctx.createGain();
        osc.type = 'triangle';
        osc.frequency.setValueAtTime(vary(380, 0.1), now);
        osc.frequency.exponentialRampToValueAtTime(80, now + 0.08);
        oscGain.gain.setValueAtTime(0.8, now);
        oscGain.gain.exponentialRampToValueAtTime(0.001, now + 0.12);
        osc.connect(oscGain);
        oscGain.connect(_sfxGain);
        osc.start(now);
        osc.stop(now + 0.13);

        // Layer 2: metallic ring (high-frequency sine, longer decay)
        const ring = ctx.createOscillator();
        const ringGain = ctx.createGain();
        ring.type = 'sine';
        ring.frequency.value = vary(1200, 0.15);
        ringGain.gain.setValueAtTime(0.3, now);
        ringGain.gain.exponentialRampToValueAtTime(0.001, now + 0.25);
        ring.connect(ringGain);
        ringGain.connect(_sfxGain);
        ring.start(now);
        ring.stop(now + 0.26);

        // Layer 3: broadband noise burst (the "crack" attack transient)
        const noiseSource = ctx.createBufferSource();
        noiseSource.buffer = createNoiseBuffer(ctx, 0.1);
        const noiseFilt = ctx.createBiquadFilter();
        noiseFilt.type = 'bandpass';
        noiseFilt.frequency.value = 2500;
        noiseFilt.Q.value = 0.8;
        const noiseGain = ctx.createGain();
        noiseGain.gain.setValueAtTime(0.6, now);
        noiseGain.gain.exponentialRampToValueAtTime(0.001, now + 0.06);
        noiseSource.connect(noiseFilt);
        noiseFilt.connect(noiseGain);
        noiseGain.connect(_sfxGain);
        noiseSource.start(now);
        noiseSource.stop(now + 0.07);
    }

    // --------------------------------------------------------
    // SFX: MAGIC SPELL (whoosh + shimmer + build)
    // --------------------------------------------------------
    // Technique: frequency-modulated sweep + noise shimmer + harmonic build
    // The rising pitch sweep signals "power gathering"; sparkle = detuned sines

    function playSpellSound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        // Main whoosh: sawtooth sweep upward, filtering
        const osc = ctx.createOscillator();
        const oscGain = ctx.createGain();
        const filter = ctx.createBiquadFilter();
        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(vary(200, 0.05), now);
        osc.frequency.exponentialRampToValueAtTime(vary(1600, 0.1), now + 0.4);
        filter.type = 'bandpass';
        filter.frequency.setValueAtTime(300, now);
        filter.frequency.exponentialRampToValueAtTime(2000, now + 0.4);
        filter.Q.value = 2;
        oscGain.gain.setValueAtTime(0, now);
        oscGain.gain.linearRampToValueAtTime(0.5, now + 0.05);
        oscGain.gain.exponentialRampToValueAtTime(0.001, now + 0.55);
        osc.connect(filter);
        filter.connect(oscGain);
        oscGain.connect(_sfxGain);
        osc.start(now);
        osc.stop(now + 0.56);

        // Shimmer: multiple detuned high-pitched sines (sparkle effect)
        const sparkleFreqs = [3200, 4000, 5100, 6400];
        sparkleFreqs.forEach((freq, i) => {
            const s = ctx.createOscillator();
            const sg = ctx.createGain();
            s.type = 'sine';
            s.frequency.value = vary(freq, 0.05);
            sg.gain.setValueAtTime(0, now + i * 0.04);
            sg.gain.linearRampToValueAtTime(0.12, now + i * 0.04 + 0.05);
            sg.gain.exponentialRampToValueAtTime(0.001, now + 0.7);
            s.connect(sg);
            sg.connect(_sfxGain);
            s.start(now + i * 0.04);
            s.stop(now + 0.71);
        });

        // Sub-bass thump at release
        const sub = ctx.createOscillator();
        const subGain = ctx.createGain();
        sub.type = 'sine';
        sub.frequency.setValueAtTime(120, now + 0.35);
        sub.frequency.exponentialRampToValueAtTime(40, now + 0.7);
        subGain.gain.setValueAtTime(0, now + 0.35);
        subGain.gain.linearRampToValueAtTime(0.4, now + 0.4);
        subGain.gain.exponentialRampToValueAtTime(0.001, now + 0.75);
        sub.connect(subGain);
        subGain.connect(_sfxGain);
        sub.start(now + 0.35);
        sub.stop(now + 0.76);
    }

    // --------------------------------------------------------
    // SFX: ARROW / PROJECTILE (whoosh + diminuendo)
    // --------------------------------------------------------
    // Technique: narrow bandpass noise sweep (simulates air pressure wave)
    // Short attack, medium decay; frequency drops as arrow passes

    function playArrowSound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        const noiseSource = ctx.createBufferSource();
        noiseSource.buffer = createNoiseBuffer(ctx, 0.3);
        noiseSource.loop = false;

        const bandpass = ctx.createBiquadFilter();
        bandpass.type = 'bandpass';
        bandpass.frequency.setValueAtTime(1800, now);
        bandpass.frequency.exponentialRampToValueAtTime(600, now + 0.25);
        bandpass.Q.value = 4;  // narrow band = "whoosh" character

        const noiseGain = ctx.createGain();
        noiseGain.gain.setValueAtTime(0, now);
        noiseGain.gain.linearRampToValueAtTime(vary(0.5, 0.1), now + 0.02);
        noiseGain.gain.exponentialRampToValueAtTime(0.001, now + 0.28);

        noiseSource.connect(bandpass);
        bandpass.connect(noiseGain);
        noiseGain.connect(_sfxGain);
        noiseSource.start(now);
        noiseSource.stop(now + 0.3);

        // Optional: brief high-pitched "zip" tone for sharper arrow
        const zip = ctx.createOscillator();
        const zipGain = ctx.createGain();
        zip.type = 'sawtooth';
        zip.frequency.setValueAtTime(2400, now);
        zip.frequency.linearRampToValueAtTime(1200, now + 0.12);
        zipGain.gain.setValueAtTime(0.15, now);
        zipGain.gain.exponentialRampToValueAtTime(0.001, now + 0.14);
        zip.connect(zipGain);
        zipGain.connect(_sfxGain);
        zip.start(now);
        zip.stop(now + 0.15);
    }

    // --------------------------------------------------------
    // SFX: UI CLICK (soft, satisfying button click)
    // --------------------------------------------------------
    // Technique: very short sine pop with fast decay
    // Keep it subtle — UI clicks should be felt, not heard

    function playClickSound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        osc.type = 'sine';
        osc.frequency.setValueAtTime(1200, now);
        osc.frequency.exponentialRampToValueAtTime(600, now + 0.02);
        gain.gain.setValueAtTime(0.3, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.04);
        osc.connect(gain);
        gain.connect(_sfxGain);
        osc.start(now);
        osc.stop(now + 0.05);
    }

    // --------------------------------------------------------
    // SFX: UNIT DEATH (descending moan + impact)
    // --------------------------------------------------------
    // Technique: pitch dive + noise thud + brief silence signal
    // The slow pitch drop = "life leaving"; noise = body hitting ground

    function playDeathSound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        // Descending cry/moan
        const osc = ctx.createOscillator();
        const oscGain = ctx.createGain();
        osc.type = 'sawtooth';
        osc.frequency.setValueAtTime(vary(600, 0.15), now);
        osc.frequency.exponentialRampToValueAtTime(60, now + 0.5);
        oscGain.gain.setValueAtTime(0, now);
        oscGain.gain.linearRampToValueAtTime(0.4, now + 0.02);
        oscGain.gain.linearRampToValueAtTime(0.1, now + 0.4);
        oscGain.gain.exponentialRampToValueAtTime(0.001, now + 0.55);

        const filter = ctx.createBiquadFilter();
        filter.type = 'lowpass';
        filter.frequency.setValueAtTime(2000, now);
        filter.frequency.linearRampToValueAtTime(300, now + 0.5);

        osc.connect(filter);
        filter.connect(oscGain);
        oscGain.connect(_sfxGain);
        osc.start(now);
        osc.stop(now + 0.56);

        // Body-fall thud (delayed noise burst)
        const thudDelay = 0.3;
        const noiseSource = ctx.createBufferSource();
        noiseSource.buffer = createNoiseBuffer(ctx, 0.2);
        const noiseFilter = ctx.createBiquadFilter();
        noiseFilter.type = 'lowpass';
        noiseFilter.frequency.value = 300;
        const noiseGain = ctx.createGain();
        noiseGain.gain.setValueAtTime(0.7, now + thudDelay);
        noiseGain.gain.exponentialRampToValueAtTime(0.001, now + thudDelay + 0.2);
        noiseSource.connect(noiseFilter);
        noiseFilter.connect(noiseGain);
        noiseGain.connect(_sfxGain);
        noiseSource.start(now + thudDelay);
        noiseSource.stop(now + thudDelay + 0.22);
    }

    // --------------------------------------------------------
    // SFX: EXPLOSION / FIREBALL
    // --------------------------------------------------------
    // Technique: white noise with dramatic lowpass sweep + sub-bass boom
    // The sub-bass is what gives explosions their "chest thump" feeling

    function playExplosionSound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        // Sub-bass boom (felt more than heard)
        const sub = ctx.createOscillator();
        const subGain = ctx.createGain();
        sub.type = 'sine';
        sub.frequency.setValueAtTime(80, now);
        sub.frequency.exponentialRampToValueAtTime(20, now + 0.4);
        subGain.gain.setValueAtTime(1.0, now);
        subGain.gain.exponentialRampToValueAtTime(0.001, now + 0.45);
        sub.connect(subGain);
        subGain.connect(_sfxGain);
        sub.start(now);
        sub.stop(now + 0.46);

        // Main noise body with sweeping lowpass
        const noiseLen = 1.5;
        const noiseSource = ctx.createBufferSource();
        noiseSource.buffer = createNoiseBuffer(ctx, noiseLen);
        noiseSource.loop = false;

        const lowpass = ctx.createBiquadFilter();
        lowpass.type = 'lowpass';
        lowpass.frequency.setValueAtTime(4000, now);
        lowpass.frequency.exponentialRampToValueAtTime(200, now + 0.8);

        const noiseGain = ctx.createGain();
        noiseGain.gain.setValueAtTime(0, now);
        noiseGain.gain.linearRampToValueAtTime(1.0, now + 0.01);  // instant attack
        noiseGain.gain.exponentialRampToValueAtTime(0.001, now + noiseLen);

        noiseSource.connect(lowpass);
        lowpass.connect(noiseGain);
        noiseGain.connect(_sfxGain);
        noiseSource.start(now);
        noiseSource.stop(now + noiseLen + 0.05);

        // Crack/pop at the very start (the ignition transient)
        const crack = ctx.createOscillator();
        const crackGain = ctx.createGain();
        crack.type = 'square';
        crack.frequency.setValueAtTime(200, now);
        crack.frequency.linearRampToValueAtTime(50, now + 0.05);
        crackGain.gain.setValueAtTime(0.8, now);
        crackGain.gain.exponentialRampToValueAtTime(0.001, now + 0.08);
        crack.connect(crackGain);
        crackGain.connect(_sfxGain);
        crack.start(now);
        crack.stop(now + 0.09);
    }

    // --------------------------------------------------------
    // SFX: VICTORY FANFARE (ascending major arpeggio + final chord)
    // --------------------------------------------------------
    // Technique: schedule oscillator notes at precise times using ctx.currentTime
    // C major pentatonic: C4 E4 G4 C5 E5 → final chord C4+E4+G4

    function playVictorySound() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        // C major arpeggio (MIDI-style: C4=261.63, E4=329.63, G4=392, C5=523.25, E5=659.25)
        const notes = [261.63, 329.63, 392.0, 523.25, 659.25];
        const noteDuration = 0.12;
        const noteSpacing = 0.13;

        notes.forEach((freq, i) => {
            const osc = ctx.createOscillator();
            const gain = ctx.createGain();
            const t = now + i * noteSpacing;
            osc.type = 'square';  // square wave = bright, clear, game-like
            osc.frequency.value = vary(freq, 0.002);  // minimal variation for melody
            gain.gain.setValueAtTime(0, t);
            gain.gain.linearRampToValueAtTime(0.25, t + 0.01);
            gain.gain.exponentialRampToValueAtTime(0.001, t + noteDuration);
            osc.connect(gain);
            gain.connect(_sfxGain);
            osc.start(t);
            osc.stop(t + noteDuration + 0.01);
        });

        // Final resolution chord (C major triad, longer sustain)
        const chordTime = now + notes.length * noteSpacing + 0.05;
        [261.63, 329.63, 392.0, 523.25].forEach(freq => {
            const osc = ctx.createOscillator();
            const gain = ctx.createGain();
            osc.type = 'triangle';
            osc.frequency.value = freq;
            gain.gain.setValueAtTime(0, chordTime);
            gain.gain.linearRampToValueAtTime(0.2, chordTime + 0.02);
            gain.gain.setValueAtTime(0.2, chordTime + 0.4);
            gain.gain.exponentialRampToValueAtTime(0.001, chordTime + 1.0);
            osc.connect(gain);
            gain.connect(_sfxGain);
            osc.start(chordTime);
            osc.stop(chordTime + 1.1);
        });
    }

    // --------------------------------------------------------
    // SFX: FOOTSTEP (low thud, organic)
    // --------------------------------------------------------

    function playFootstep() {
        const ctx = getCtx();
        if (!ctx) return;
        const now = ctx.currentTime;

        // Low-frequency thud (the heel impact)
        const osc = ctx.createOscillator();
        const oscGain = ctx.createGain();
        osc.type = 'sine';
        osc.frequency.setValueAtTime(vary(160, 0.2), now);
        osc.frequency.exponentialRampToValueAtTime(50, now + 0.08);
        oscGain.gain.setValueAtTime(0.5, now);
        oscGain.gain.exponentialRampToValueAtTime(0.001, now + 0.1);
        osc.connect(oscGain);
        oscGain.connect(_sfxGain);
        osc.start(now);
        osc.stop(now + 0.11);

        // Brief high-freq noise (surface texture: dirt/grass)
        const noiseSource = ctx.createBufferSource();
        noiseSource.buffer = createNoiseBuffer(ctx, 0.08);
        const filt = ctx.createBiquadFilter();
        filt.type = 'bandpass';
        filt.frequency.value = vary(800, 0.3);
        filt.Q.value = 2;
        const noiseGain = ctx.createGain();
        noiseGain.gain.setValueAtTime(0.2, now);
        noiseGain.gain.exponentialRampToValueAtTime(0.001, now + 0.07);
        noiseSource.connect(filt);
        filt.connect(noiseGain);
        noiseGain.connect(_sfxGain);
        noiseSource.start(now);
        noiseSource.stop(now + 0.09);
    }

    // --------------------------------------------------------
    // PERCUSSION SYNTHESIZERS (for ambient music drums)
    // --------------------------------------------------------

    function playKick(time) {
        const ctx = getCtx();
        if (!ctx) return;
        const t = time || ctx.currentTime;

        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        osc.type = 'triangle';
        osc.frequency.setValueAtTime(120, t);
        osc.frequency.exponentialRampToValueAtTime(20, t + 0.4);
        gain.gain.setValueAtTime(1.0, t);
        gain.gain.exponentialRampToValueAtTime(0.001, t + 0.4);
        osc.connect(gain);
        gain.connect(_musicGain);
        osc.start(t);
        osc.stop(t + 0.41);
    }

    function playSnare(time) {
        const ctx = getCtx();
        if (!ctx) return;
        const t = time || ctx.currentTime;

        // Noise component
        const noiseSource = ctx.createBufferSource();
        noiseSource.buffer = createNoiseBuffer(ctx, 0.2);
        const noiseFilter = ctx.createBiquadFilter();
        noiseFilter.type = 'highpass';
        noiseFilter.frequency.value = 1000;
        const noiseGain = ctx.createGain();
        noiseGain.gain.setValueAtTime(0.8, t);
        noiseGain.gain.exponentialRampToValueAtTime(0.001, t + 0.2);
        noiseSource.connect(noiseFilter);
        noiseFilter.connect(noiseGain);
        noiseGain.connect(_musicGain);
        noiseSource.start(t);
        noiseSource.stop(t + 0.21);

        // Tone component (body)
        const tone = ctx.createOscillator();
        const toneGain = ctx.createGain();
        tone.type = 'triangle';
        tone.frequency.value = 200;
        toneGain.gain.setValueAtTime(0.4, t);
        toneGain.gain.exponentialRampToValueAtTime(0.001, t + 0.1);
        tone.connect(toneGain);
        toneGain.connect(_musicGain);
        tone.start(t);
        tone.stop(t + 0.11);
    }

    function playHiHat(time, open) {
        const ctx = getCtx();
        if (!ctx) return;
        const t = time || ctx.currentTime;
        const decay = open ? 0.3 : 0.05;

        // Hi-hat = 6 square waves at metallic frequency ratios
        const fundamental = 40;
        const ratios = [2, 3, 4.16, 5.43, 6.79, 8.21];
        const masterGain = ctx.createGain();
        const bandpass = ctx.createBiquadFilter();
        const highpass = ctx.createBiquadFilter();
        bandpass.type = 'bandpass';
        bandpass.frequency.value = 10000;
        highpass.type = 'highpass';
        highpass.frequency.value = 7000;
        masterGain.gain.setValueAtTime(0.3, t);
        masterGain.gain.exponentialRampToValueAtTime(0.001, t + decay);
        bandpass.connect(highpass);
        highpass.connect(masterGain);
        masterGain.connect(_musicGain);

        ratios.forEach(ratio => {
            const osc = ctx.createOscillator();
            osc.type = 'square';
            osc.frequency.value = fundamental * ratio;
            osc.connect(bandpass);
            osc.start(t);
            osc.stop(t + decay + 0.01);
        });
    }

    // --------------------------------------------------------
    // AMBIENT MUSIC (Procedural fantasy loop)
    // --------------------------------------------------------
    // Strategy: pentatonic scale + pad drones + slow chord progression
    // Pentatonic avoids dissonance — any random combination sounds OK
    // Uses Web Audio scheduler pattern (lookahead with setTimeout)
    // --------------------------------------------------------

    const PENTATONIC_C = [
        261.63,  // C4
        293.66,  // D4
        329.63,  // E4
        392.00,  // G4
        440.00,  // A4
        523.25,  // C5
        587.33,  // D5
        659.25,  // E5
    ];

    // Medieval/fantasy chord progression: Am → F → C → G (in pentatonic-friendly voicings)
    const CHORD_PROG = [
        [220.00, 261.63, 329.63],  // A minor (A3 C4 E4)
        [174.61, 220.00, 261.63],  // F major  (F3 A3 C4)
        [261.63, 329.63, 392.00],  // C major  (C4 E4 G4)
        [196.00, 246.94, 293.66],  // G major  (G3 B3 D4)
    ];

    let _musicScheduler = null;
    let _musicPlaying = false;
    let _musicBeat = 0;
    let _musicTick = 0;
    let _chordIndex = 0;
    let _padNodes = [];

    const MUSIC_TEMPO = 90;  // BPM
    const LOOKAHEAD_MS = 50;
    const SCHEDULE_AHEAD_S = 0.15;

    // Create a sustained pad chord (strings/organ-like)
    function schedulePadChord(freqs, startTime, duration) {
        const ctx = getCtx();
        freqs.forEach(freq => {
            // Each chord note: triangle + slight detune for warmth
            [0, 2, -2].forEach(detuneCents => {
                const osc = ctx.createOscillator();
                const gain = ctx.createGain();
                const filter = ctx.createBiquadFilter();
                osc.type = 'triangle';
                osc.frequency.value = freq * Math.pow(2, detuneCents / 1200);
                filter.type = 'lowpass';
                filter.frequency.value = 800;
                filter.Q.value = 0.5;
                // Slow attack (pad-like)
                gain.gain.setValueAtTime(0, startTime);
                gain.gain.linearRampToValueAtTime(0.08, startTime + 0.8);
                gain.gain.setValueAtTime(0.08, startTime + duration - 0.6);
                gain.gain.linearRampToValueAtTime(0, startTime + duration);
                osc.connect(filter);
                filter.connect(gain);
                gain.connect(_musicGain);
                osc.start(startTime);
                osc.stop(startTime + duration + 0.1);
                _padNodes.push(osc);
            });
        });
    }

    // Schedule a single melody note from the pentatonic scale
    function scheduleMelodyNote(time, noteIndex) {
        const ctx = getCtx();
        const freq = PENTATONIC_C[noteIndex % PENTATONIC_C.length];
        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        osc.type = 'square';
        osc.frequency.value = freq;
        gain.gain.setValueAtTime(0, time);
        gain.gain.linearRampToValueAtTime(0.06, time + 0.02);
        gain.gain.exponentialRampToValueAtTime(0.001, time + 0.25);
        osc.connect(gain);
        gain.connect(_musicGain);
        osc.start(time);
        osc.stop(time + 0.27);
        _padNodes.push(osc);
    }

    let _nextNoteTime = 0;
    let _currentBeat = 0;
    let _currentBar = 0;

    // 16-step drum pattern (1 = play, 0 = silent):
    // Format: [kick, snare, hihat] per 16th note
    const DRUM_PATTERN = [
        // 0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
        [1, 0, 0, 0,  0, 0, 0, 0,  1, 0, 0, 0,  0, 0, 1, 0], // kick
        [0, 0, 0, 0,  1, 0, 0, 0,  0, 0, 0, 0,  1, 0, 0, 0], // snare
        [1, 0, 1, 0,  1, 0, 1, 0,  1, 0, 1, 0,  1, 0, 1, 0], // hihat
    ];

    // Melody pattern: which pentatonic scale degrees to play (−1 = rest)
    const MELODY_PATTERN = [-1, -1, -1, 5, -1, -1, 3, -1, -1, -1, 4, -1, 5, -1, -1, -1];

    function _scheduleMusicStep(time, step) {
        const ctx = getCtx();
        const secondsPerBeat = 60.0 / MUSIC_TEMPO;
        const sixteenth = secondsPerBeat / 4;
        const t = time + step * sixteenth;

        if (DRUM_PATTERN[0][step]) playKick(t);
        if (DRUM_PATTERN[1][step]) playSnare(t);
        if (DRUM_PATTERN[2][step]) playHiHat(t, false);

        // Melody
        if (MELODY_PATTERN[step] >= 0) {
            scheduleMelodyNote(t, MELODY_PATTERN[step]);
        }
    }

    function _musicLoop() {
        if (!_musicPlaying) return;
        const ctx = getCtx();
        const secondsPerBeat = 60.0 / MUSIC_TEMPO;
        const secondsPerBar = secondsPerBeat * 4;  // 4/4 time

        while (_nextNoteTime < ctx.currentTime + SCHEDULE_AHEAD_S) {
            const step = _currentBeat % 16;

            // Schedule chord pad every 4 beats (once per bar)
            if (step === 0) {
                schedulePadChord(
                    CHORD_PROG[_chordIndex % CHORD_PROG.length],
                    _nextNoteTime,
                    secondsPerBar
                );
                _chordIndex++;
            }

            _scheduleMusicStep(_nextNoteTime - (step * (secondsPerBeat / 4)), step);

            _nextNoteTime += secondsPerBeat / 4;  // advance by one 16th note
            _currentBeat++;
        }

        _musicScheduler = setTimeout(_musicLoop, LOOKAHEAD_MS);
    }

    function playAmbientMusic() {
        if (_musicPlaying) return;
        const ctx = getCtx();
        if (!ctx) return;

        _musicPlaying = true;
        _currentBeat = 0;
        _chordIndex = 0;
        _padNodes = [];
        _nextNoteTime = ctx.currentTime + 0.1;

        _musicLoop();
    }

    function stopAmbientMusic(fadeOutSeconds) {
        _musicPlaying = false;
        if (_musicScheduler) clearTimeout(_musicScheduler);
        _musicScheduler = null;

        // Stop all pad nodes
        if (_musicGain && fadeOutSeconds) {
            fadeMusic(0, fadeOutSeconds);
            setTimeout(() => {
                _padNodes.forEach(n => { try { n.stop(); } catch(e) {} });
                _padNodes = [];
            }, fadeOutSeconds * 1000 + 50);
        } else {
            _padNodes.forEach(n => { try { n.stop(); } catch(e) {} });
            _padNodes = [];
        }
    }

    // --------------------------------------------------------
    // SOUND POOLING (for rapid repeated hits)
    // --------------------------------------------------------
    // The Web Audio API natively supports polyphony — every call to playHitSound()
    // creates independent nodes and plays simultaneously. No explicit pool needed.
    // However, you should limit simultaneous identical sounds to prevent clipping:

    const _soundCooldowns = {};

    function withCooldown(name, minIntervalMs, fn) {
        const now = Date.now();
        const last = _soundCooldowns[name] || 0;
        if (now - last < minIntervalMs) return;
        _soundCooldowns[name] = now;
        fn();
    }

    // --------------------------------------------------------
    // PUBLIC API
    // --------------------------------------------------------

    return {
        init,
        // Volume
        setMasterVolume,
        setMusicVolume,
        setSfxVolume,
        fadeMusic,
        // SFX
        playHit:       () => withCooldown('hit', 40, playHitSound),
        playSpell:     () => withCooldown('spell', 100, playSpellSound),
        playArrow:     () => withCooldown('arrow', 50, playArrowSound),
        playClick:     () => withCooldown('click', 20, playClickSound),
        playDeath:     () => withCooldown('death', 150, playDeathSound),
        playExplosion: () => withCooldown('explosion', 200, playExplosionSound),
        playVictory:   playVictorySound,
        playFootstep:  () => withCooldown('step', 80, playFootstep),
        // Music
        playAmbientMusic,
        stopAmbientMusic,
        // Direct access for custom sounds
        getCtx,
        createNoiseBuffer,
    };
})();

// ============================================================
// USAGE — embed in any game HTML
// ============================================================
//
// 1. Include the GameAudio block above in a <script> tag.
//
// 2. Initialize on first user interaction:
//    document.addEventListener('click', () => GameAudio.init(), { once: true });
//
// 3. Trigger sounds from game events:
//    unitAttacks()      → GameAudio.playHit()
//    spellCast()        → GameAudio.playSpell()
//    arrowFired()       → GameAudio.playArrow()
//    unitDies()         → GameAudio.playDeath()
//    buildingExplodes() → GameAudio.playExplosion()
//    buttonClicked()    → GameAudio.playClick()
//    gameWon()          → GameAudio.playVictory()
//
// 4. Start/stop ambient music:
//    GameAudio.playAmbientMusic()
//    GameAudio.stopAmbientMusic(2.0)  // 2-second fade out
//
// 5. Volume sliders (0.0 – 1.0):
//    GameAudio.setMasterVolume(0.8)
//    GameAudio.setMusicVolume(0.4)
//    GameAudio.setSfxVolume(1.0)
```

---

## 4. Integration Pattern for AI-Generated Games

When an AI agent generates an HTML game, include this boilerplate:

```html
<script>
// [PASTE GameAudio library here — the entire IIFE block]

// Auto-initialize on any click anywhere on the page
(function() {
    let initialized = false;
    function tryInit(e) {
        if (initialized) return;
        initialized = true;
        GameAudio.init();
        // Start ambient music immediately after first interaction
        GameAudio.playAmbientMusic();
    }
    document.addEventListener('click', tryInit);
    document.addEventListener('touchstart', tryInit);
    document.addEventListener('keydown', tryInit);
})();
</script>
```

Then in the game event handlers:
```javascript
// In combat resolution:
function resolveAttack(attacker, defender) {
    const damage = calculateDamage(attacker, defender);
    if (damage > 0) {
        if (attacker.type === 'archer') GameAudio.playArrow();
        else if (attacker.type === 'mage') GameAudio.playSpell();
        else GameAudio.playHit();
    }
    if (defender.hp <= 0) {
        GameAudio.playDeath();
    }
}

// In UI handlers:
document.querySelectorAll('button').forEach(btn => {
    btn.addEventListener('click', () => GameAudio.playClick());
});

// On game over:
function showVictory() {
    GameAudio.stopAmbientMusic(1.5);
    setTimeout(() => GameAudio.playVictory(), 300);
}
```

---

## 5. Gotchas and Common Mistakes

### The Big Ones

**1. Forgetting to resume the AudioContext.**
On Chrome and Safari, the context starts in `suspended` state. Sounds created before
a user gesture silently fail. Always check `ctx.state === 'suspended'` before playing.

**2. Trying to restart a stopped OscillatorNode.**
`OscillatorNode.start()` can only be called once. After `stop()`, the node is dead.
Always create a fresh node for each sound event.

**3. Pure sine waves with no envelope.**
Instant on/off creates audible clicks (Gibbs phenomenon). Always ramp gain to 0
instead of cutting abruptly. The minimum is:
```javascript
gain.gain.setValueAtTime(0.5, now);
gain.gain.exponentialRampToValueAtTime(0.001, now + 0.1); // NOT .value = 0
```

**4. Not using exponential ramps for amplitude.**
`linearRamp` sounds mechanical. `exponentialRamp` matches human volume perception
(decibel scale is logarithmic). Exception: pitch glides often sound better linear.

**5. No DynamicsCompressor on master.**
Without compression, multiple simultaneous sounds clip badly. The compressor at the
end of the chain is what makes everything sound "glued together" and not distorted.

**6. Memory leaks from lingering nodes.**
AudioBuffer nodes auto-garbage-collect after `stop()`. OscillatorNodes do too.
But if you're building scheduled music with `_padNodes` arrays, always clear them
when stopping music or the browser accumulates dead references.

**7. Scheduling with setTimeout instead of AudioContext time.**
`setTimeout(playNote, 500)` drifts badly under load. Always schedule future sounds
using `ctx.currentTime + offset`. The lookahead scheduler pattern (schedule ahead
by 150ms, poll every 50ms) is the correct approach for music.

**8. No pitch variation.**
Identical sounds on every hit sound robotic. Add ±5-10% random variation to
frequency values: `freq * (0.95 + Math.random() * 0.1)`.

**9. Forgetting iOS volume restrictions.**
On iOS, programmatic volume control (`HTMLMediaElement.volume`) is disabled.
Use Web Audio GainNodes exclusively for all volume control.

**10. ScriptProcessorNode for noise generation.**
`ScriptProcessorNode` is deprecated and runs on the main thread (causes stuttering).
Use pre-generated `AudioBuffer` with random data instead — much more efficient.

---

## 6. Quality Checklist for Generated Games

Before considering game audio "done," verify:

- [ ] AudioContext initializes only after user gesture (no autoplay errors in console)
- [ ] DynamicsCompressor on master output (prevents clipping)
- [ ] Each sound has proper amplitude envelope (no abrupt cuts)
- [ ] Hit sounds use pitch sweep (not static frequency)
- [ ] Mix has noise layer for impacts (not just pure tones)
- [ ] Music uses pentatonic or diatonic scale (no random dissonance)
- [ ] Music loops seamlessly (scheduler runs continuously)
- [ ] Volume controls work for music/sfx independently
- [ ] Rapid repeated sounds (melee) don't clip (cooldown or polyphony limit)
- [ ] No console errors about suspended AudioContext

---

## 7. Reference Architecture for Game Prompt Injection

When injecting into a game generation prompt, use this template snippet:

```
Include a complete Web Audio synthesis sound system with these sounds:
- GameAudio.playHit() — sword/melee impact
- GameAudio.playSpell() — magic cast
- GameAudio.playArrow() — ranged attack
- GameAudio.playClick() — UI button
- GameAudio.playDeath() — unit dies
- GameAudio.playExplosion() — area attack
- GameAudio.playVictory() — win condition
- GameAudio.playAmbientMusic() — looping fantasy background
- GameAudio.stopAmbientMusic(fadeSeconds) — fade out music

All sounds must be synthesized with Web Audio API only. No external audio files.
Trigger sounds on game events. Initialize AudioContext on first user interaction.
Include a DynamicsCompressor on the master output.
```

---

## Sources

- [MDN: Audio for Web Games](https://developer.mozilla.org/en-US/docs/Games/Techniques/Audio_for_Web_Games)
- [MDN: Web Audio API Advanced Techniques](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Advanced_techniques)
- [MDN: Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [web.dev: Developing game audio with the Web Audio API](https://web.dev/webaudio-games/)
- [Sonoport: Synthesising Sounds with Web Audio API](https://sonoport.github.io/synthesising-sounds-webaudio.html)
- [noisehack: How to Generate Noise with the Web Audio API](https://noisehack.com/generate-noise-web-audio-api/)
- [GitHub: KilledByAPixel/ZzFX — Tiny Sound FX System](https://github.com/KilledByAPixel/ZzFX)
- [GitHub: web-audio-components/simple-reverb](https://github.com/web-audio-components/simple-reverb)
- [GitHub: adelespinasse/reverbGen](https://github.com/adelespinasse/reverbGen)
- [Synthesizing Hi-Hats with Web Audio](http://joesul.li/van/synthesizing-hi-hats/)
- [Dynamic Music in Games using WebAudio](https://cschnack.de/blog/2020/webaudio/)
