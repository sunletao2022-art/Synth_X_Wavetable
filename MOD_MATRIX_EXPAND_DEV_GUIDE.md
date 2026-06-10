# Mod Matrix Destination Expansion — Dev Guide
## For Codex Execution

**Goal:** Expand mod matrix destinations from 27 → 49.
All 15 sources (feg, env1–3, lfo1–3, vel, note, step, cc1, mpe_pressure, mpe_timbre,
mpe_pb, pb) will gain routes to every new destination.
Total depth params: 15 × 49 = **735** (was 405).

---

## Destination Index Table (complete)

```
Idx  ID            Name                    Unit / Range
--- OSC 1 (existing 0-8) ---
  0  osc1pos       OSC1 Position           0–100 %
  1  osc1tune      OSC1 Tune               ±36 st
  2  osc1fine      OSC1 Fine Tune          ±100 ct
  3  osc1level     OSC1 Level              -100–0 dB
  4  osc1pan       OSC1 Pan                -1–1
  5  osc1detune    OSC1 Detune             0–0.5
  6  osc1spread    OSC1 Spread             ±100 %
  7  osc1formant   OSC1 Formant            -1–1
  8  osc1bend      OSC1 Bend               -1–1
--- OSC 2 (existing 9-17) ---
  9  osc2pos       OSC2 Position           0–100 %
 10  osc2tune      OSC2 Tune               ±36 st
 11  osc2fine      OSC2 Fine Tune          ±100 ct
 12  osc2level     OSC2 Level              -100–0 dB
 13  osc2pan       OSC2 Pan                -1–1
 14  osc2detune    OSC2 Detune             0–0.5
 15  osc2spread    OSC2 Spread             ±100 %
 16  osc2formant   OSC2 Formant            -1–1
 17  osc2bend      OSC2 Bend               -1–1
--- Sub / Noise (existing 18-22) ---
 18  subtune       Sub Tune                ±36 st
 19  sublevel      Sub Level               -100–0 dB
 20  subpan        Sub Pan                 -1–1
 21  noiselevel    Noise Level             -100–0 dB
 22  noisepan      Noise Pan               -1–1
--- Filter (existing 23-25, extended 26-30) ---
 23  fltfreq       Filter Cutoff           MIDI note 0–maxFreq
 24  fltres        Filter Resonance        0–100
 25  fltamount     Filter EG Amount        -1–1
 26  fltkey        Filter Key Tracking     0–100 %  ← NEW
 27  fltvel        Filter Vel Tracking     0–100 %  ← NEW
--- Filter EG times (NEW 28-31) ---
 28  fltattack     Filter EG Attack        0–60 s   ← NEW
 29  fltdecay      Filter EG Decay         0–60 s   ← NEW
 30  fltsustain    Filter EG Sustain       0–100 %  ← NEW
 31  fltrelease    Filter EG Release       0–60 s   ← NEW
--- LFO 1 (NEW 32-35) ---
 32  lfo1rate      LFO 1 Rate              0–50 Hz  ← NEW
 33  lfo1depth     LFO 1 Depth             -1–1     ← NEW
 34  lfo1phase     LFO 1 Phase             -1–1     ← NEW
 35  lfo1offset    LFO 1 Offset            -1–1     ← NEW
--- LFO 2 (NEW 36-39) ---
 36  lfo2rate      LFO 2 Rate              0–50 Hz  ← NEW
 37  lfo2depth     LFO 2 Depth             -1–1     ← NEW
 38  lfo2phase     LFO 2 Phase             -1–1     ← NEW
 39  lfo2offset    LFO 2 Offset            -1–1     ← NEW
--- LFO 3 (NEW 40-43) ---
 40  lfo3rate      LFO 3 Rate              0–50 Hz  ← NEW
 41  lfo3depth     LFO 3 Depth             -1–1     ← NEW
 42  lfo3phase     LFO 3 Phase             -1–1     ← NEW
 43  lfo3offset    LFO 3 Offset            -1–1     ← NEW
--- Amp EG times (NEW 44-47) ---
 44  ampattack     Amp Attack              0–60 s   ← NEW
 45  ampdecay      Amp Decay               0–60 s   ← NEW
 46  ampsustain    Amp Sustain             0–100 %  ← NEW
 47  amprelease    Amp Release             0–60 s   ← NEW
--- Global (existing, moved from 26) ---
 48  masterlevel   Master Level            -100–0 dB  (was index 26)
```

> **Note:** `masterlevel` moved from index 26 → 48 because new entries were inserted
> before it. All dstIdx references in `WavetableVoice.cpp` must use the new indices.

---

## 1. `plugin/Source/PluginProcessor.h`

### 1.1 Change `numDsts`

Find and replace inside `struct ModDepthParams`:

```cpp
// BEFORE:
static constexpr int numDsts  = 27;

// AFTER:
static constexpr int numDsts  = 49;
```

---

## 2. `plugin/Source/PluginProcessor.cpp`

### 2.1 Replace `kModDstIds` array

Find the anonymous namespace block that contains `kModDstIds` and replace it entirely:

```cpp
// REPLACE the entire block:
//   static constexpr int kNumModDsts = ...
//   static const char* kModDstIds[kNumModDsts] = { ... };
//   static const char* kModDstNames[kNumModDsts] = { ... };
// WITH:

static constexpr int kNumModDsts = 49;

static const char* kModDstIds[kNumModDsts] = {
    // OSC 1 (0-8)
    "osc1pos","osc1tune","osc1fine","osc1level","osc1pan",
    "osc1detune","osc1spread","osc1formant","osc1bend",
    // OSC 2 (9-17)
    "osc2pos","osc2tune","osc2fine","osc2level","osc2pan",
    "osc2detune","osc2spread","osc2formant","osc2bend",
    // Sub / Noise (18-22)
    "subtune","sublevel","subpan","noiselevel","noisepan",
    // Filter (23-31)
    "fltfreq","fltres","fltamount","fltkey","fltvel",
    "fltattack","fltdecay","fltsustain","fltrelease",
    // LFO 1 (32-35)
    "lfo1rate","lfo1depth","lfo1phase","lfo1offset",
    // LFO 2 (36-39)
    "lfo2rate","lfo2depth","lfo2phase","lfo2offset",
    // LFO 3 (40-43)
    "lfo3rate","lfo3depth","lfo3phase","lfo3offset",
    // Amp EG (44-47)
    "ampattack","ampdecay","ampsustain","amprelease",
    // Global (48)
    "masterlevel"
};

static const char* kModDstNames[kNumModDsts] = {
    // OSC 1
    "OSC1 Pos","OSC1 Tune","OSC1 Fine","OSC1 Level","OSC1 Pan",
    "OSC1 Detune","OSC1 Spread","OSC1 Formant","OSC1 Bend",
    // OSC 2
    "OSC2 Pos","OSC2 Tune","OSC2 Fine","OSC2 Level","OSC2 Pan",
    "OSC2 Detune","OSC2 Spread","OSC2 Formant","OSC2 Bend",
    // Sub / Noise
    "Sub Tune","Sub Level","Sub Pan","Noise Level","Noise Pan",
    // Filter
    "Flt Freq","Flt Res","Flt Amount","Flt Key","Flt Vel",
    "Flt Atk","Flt Dcy","Flt Sus","Flt Rel",
    // LFO 1
    "LFO1 Rate","LFO1 Depth","LFO1 Phase","LFO1 Offset",
    // LFO 2
    "LFO2 Rate","LFO2 Depth","LFO2 Phase","LFO2 Offset",
    // LFO 3
    "LFO3 Rate","LFO3 Depth","LFO3 Phase","LFO3 Offset",
    // Amp EG
    "Amp Atk","Amp Dcy","Amp Sus","Amp Rel",
    // Global
    "Master Level"
};
```

---

## 3. `plugin/Source/WavetableVoice.cpp` — Apply new destinations

All changes are inside `WavetableVoice::updateParams()`.

### 3.1 Collect modulator outputs — unchanged

The `modSrcOut[15]` array and `modSum` lambda are unchanged. `modSum(dstIdx)` sums
all 15 source contributions for any destination index.

### 3.2 Filter section — add key, vel, and EG time modulation

Find the existing block that starts with:
```cpp
float n = getValue (proc.filterParams.frequency);
n += (currentlyPlayingNote.initialNote - 60) * getValue (proc.filterParams.keyTracking);
n += filterEnv * filterSens * getValue (proc.filterParams.amount) * filterWidth;
```

Replace with:
```cpp
float n = getValue (proc.filterParams.frequency);
n += modSum (23) * filterWidth;   // dstIdx 23 = fltfreq

// Key tracking with explicit mod (dstIdx 26 = fltkey, range 0-100)
float keyTrack = juce::jlimit (0.0f, 100.0f,
    getValue (proc.filterParams.keyTracking) + modSum (26) * 100.0f);
n += (currentlyPlayingNote.initialNote - 60) * (keyTrack / 100.0f);

// Filter EG amount with explicit mod (dstIdx 25 = fltamount)
float fltAmtMod = juce::jlimit (-1.0f, 1.0f,
    getValue (proc.filterParams.amount) + modSum (25));

// Velocity sensitivity with explicit mod (dstIdx 27 = fltvel, range 0-100)
float fltVelMod = juce::jlimit (0.0f, 100.0f,
    getValue (proc.filterParams.velocityTracking) + modSum (27) * 100.0f);
float filterSens = currentlyPlayingNote.noteOnVelocity.asUnsignedFloat()
                 * (fltVelMod / 100.0f)
                 + 1.0f - (fltVelMod / 100.0f);

n += filterEnv * filterSens * fltAmtMod * filterWidth;
```

Then find where filter EG ADSR is set:
```cpp
filterADSR.setAttack  (getValue (proc.filterParams.attack));
filterADSR.setSustainLevel (getValue (proc.filterParams.sustain));
filterADSR.setDecay   (getValue (proc.filterParams.decay));
filterADSR.setRelease (getValue (proc.filterParams.release));
```

Replace with:
```cpp
// Filter EG ADSR with explicit mod (dstIdx 28-31)
filterADSR.setAttack  (juce::jlimit (0.0f, 60.0f,
    getValue (proc.filterParams.attack)  + modSum (28) * 60.0f));
filterADSR.setDecay   (juce::jlimit (0.0f, 60.0f,
    getValue (proc.filterParams.decay)   + modSum (29) * 60.0f));
filterADSR.setSustainLevel (juce::jlimit (0.0f, 100.0f,
    getValue (proc.filterParams.sustain) + modSum (30) * 100.0f));
filterADSR.setRelease (juce::jlimit (0.0f, 60.0f,
    getValue (proc.filterParams.release) + modSum (31) * 60.0f));
```

Also replace how resonance is computed (find the existing `float q = gin::Q / ...` line):
```cpp
// Resonance with explicit mod (dstIdx 24 = fltres)
float resBase = juce::jlimit (0.0f, 100.0f,
    getValue (proc.filterParams.resonance) + modSum (24) * 100.0f);
float q = gin::Q / (1.0f - (resBase / 100.0f) * 0.99f);
```

### 3.3 LFO section — add rate, depth, phase, offset modulation

Find the LFO loop that contains:
```cpp
params.waveShape = (gin::LFO::WaveShape) int (proc.lfoParams[i].wave->getProcValue());
params.frequency = freq;
params.phase     = getValue (proc.lfoParams[i].phase);
params.offset    = getValue (proc.lfoParams[i].offset);
params.depth     = getValue (proc.lfoParams[i].depth);
params.delay     = getValue (proc.lfoParams[i].delay);
params.fade      = getValue (proc.lfoParams[i].fade);
```

Replace with (keeping waveShape, delay, fade unchanged):
```cpp
params.waveShape = (gin::LFO::WaveShape) int (proc.lfoParams[i].wave->getProcValue());

// Destination indices: lfo_n_rate  = 32 + i*4
//                      lfo_n_depth = 33 + i*4
//                      lfo_n_phase = 34 + i*4
//                      lfo_n_offset= 35 + i*4
const int lfoBase = 32 + i * 4;

params.frequency = juce::jlimit (0.0f, 50.0f, freq + modSum (lfoBase + 0) * 50.0f);
params.depth     = juce::jlimit (-1.0f, 1.0f,
    getValue (proc.lfoParams[i].depth)  + modSum (lfoBase + 1));
params.phase     = juce::jlimit (-1.0f, 1.0f,
    getValue (proc.lfoParams[i].phase)  + modSum (lfoBase + 2));
params.offset    = juce::jlimit (-1.0f, 1.0f,
    getValue (proc.lfoParams[i].offset) + modSum (lfoBase + 3));
params.delay     = getValue (proc.lfoParams[i].delay);
params.fade      = getValue (proc.lfoParams[i].fade);
```

### 3.4 Amp EG — add ADSR time modulation

Find the existing amp EG block:
```cpp
adsr.setAttack      (getValue (proc.adsrParams.attack));
adsr.setDecay       (getValue (proc.adsrParams.decay));
adsr.setSustainLevel(getValue (proc.adsrParams.sustain));
adsr.setRelease     (fastKill ? 0.01f : getValue (proc.adsrParams.release));
```

Replace with:
```cpp
// Amp EG ADSR with explicit mod (dstIdx 44-47)
adsr.setAttack      (juce::jlimit (0.0f, 60.0f,
    getValue (proc.adsrParams.attack)  + modSum (44) * 60.0f));
adsr.setDecay       (juce::jlimit (0.0f, 60.0f,
    getValue (proc.adsrParams.decay)   + modSum (45) * 60.0f));
adsr.setSustainLevel(juce::jlimit (0.0f, 100.0f,
    getValue (proc.adsrParams.sustain) + modSum (46) * 100.0f));
adsr.setRelease     (fastKill ? 0.01f :
    juce::jlimit (0.0f, 60.0f,
        getValue (proc.adsrParams.release) + modSum (47) * 60.0f));
```

### 3.5 Master level — update dstIdx from 26 → 48

Search in `WavetableVoice.cpp` for any `modSum(26)` that was applied to master level
and change it to `modSum(48)`. If the master level mod was applied in
`PluginProcessor.cpp`'s `processBlock()` instead, update that reference there.

---

## 4. Build & Verify

```bash
cd /path/to/Synth_X/Wavetable

cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --config Debug --target Wavetable_VST3 -j$(sysctl -n hw.logicalcpu)
cp -r build/Wavetable_artefacts/Debug/VST3/Wavetable.vst3 \
      ~/Library/Audio/Plug-Ins/VST3/Wavetable.vst3

# Verify param count
python3 -c "
from pedalboard import load_plugin
p = load_plugin('$HOME/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
mod = [k for k in p.parameters if k.startswith('mod_')]
print('Total params:', len(p.parameters))
print('Mod depth params:', len(mod))
# Expected: 735 mod params (15 × 49)
# Spot-check new destinations:
new_dsts = ['lfo1rate','lfo1depth','lfo2rate','fltkey','fltvel',
            'fltattack','ampattack','ampdecay']
for d in new_dsts:
    sample = f'mod_feg_{d}'
    found = sample in p.parameters
    print(f'  {sample}: {\"OK\" if found else \"MISSING\"}')
"
```

---

## 5. Python Schema Update (`app/schemas/wavetable_request.py`)

After rebuilding, add these fields to `WavetableSoundRequest`.
Add them after the existing `mod_note_flt_freq` field:

```python
# ── Mod matrix — Filter key/vel tracking ─────────────────────────────────────
mod_filter_eg_flt_key: float    = Field(default=0., ge=-1., le=1.)
mod_filter_eg_flt_vel: float    = Field(default=0., ge=-1., le=1.)
mod_env_1_flt_key: float        = Field(default=0., ge=-1., le=1.)
mod_env_1_flt_vel: float        = Field(default=0., ge=-1., le=1.)
mod_lfo_1_flt_key: float        = Field(default=0., ge=-1., le=1.)
mod_velocity_flt_key: float     = Field(default=0., ge=-1., le=1.)
mod_note_flt_key: float         = Field(default=0., ge=-1., le=1.)

# ── Mod matrix — Filter EG times ──────────────────────────────────────────────
# (use filter_eg, env_1–3, lfo_1–3, velocity as sources)
mod_filter_eg_flt_attack: float  = Field(default=0., ge=-1., le=1.)
mod_filter_eg_flt_decay: float   = Field(default=0., ge=-1., le=1.)
mod_filter_eg_flt_sustain: float = Field(default=0., ge=-1., le=1.)
mod_filter_eg_flt_release: float = Field(default=0., ge=-1., le=1.)
mod_env_1_flt_attack: float      = Field(default=0., ge=-1., le=1.)
mod_env_1_flt_decay: float       = Field(default=0., ge=-1., le=1.)
mod_velocity_flt_attack: float   = Field(default=0., ge=-1., le=1.)
mod_velocity_flt_decay: float    = Field(default=0., ge=-1., le=1.)

# ── Mod matrix — LFO 1 rate/depth/phase/offset ───────────────────────────────
mod_filter_eg_lfo1_rate: float   = Field(default=0., ge=-1., le=1., description="Filter EG → LFO 1 rate")
mod_filter_eg_lfo1_depth: float  = Field(default=0., ge=-1., le=1.)
mod_env_1_lfo1_rate: float       = Field(default=0., ge=-1., le=1.)
mod_env_1_lfo1_depth: float      = Field(default=0., ge=-1., le=1.)
mod_lfo_2_lfo1_rate: float       = Field(default=0., ge=-1., le=1., description="LFO 2 → LFO 1 rate (LFO FM)")
mod_lfo_2_lfo1_depth: float      = Field(default=0., ge=-1., le=1.)
mod_velocity_lfo1_rate: float    = Field(default=0., ge=-1., le=1.)
mod_velocity_lfo1_depth: float   = Field(default=0., ge=-1., le=1.)
mod_note_lfo1_rate: float        = Field(default=0., ge=-1., le=1.)

# ── Mod matrix — LFO 2 rate/depth ────────────────────────────────────────────
mod_env_1_lfo2_rate: float       = Field(default=0., ge=-1., le=1.)
mod_env_1_lfo2_depth: float      = Field(default=0., ge=-1., le=1.)
mod_lfo_1_lfo2_rate: float       = Field(default=0., ge=-1., le=1.)
mod_velocity_lfo2_rate: float    = Field(default=0., ge=-1., le=1.)

# ── Mod matrix — LFO 3 rate/depth ────────────────────────────────────────────
mod_env_1_lfo3_rate: float       = Field(default=0., ge=-1., le=1.)
mod_velocity_lfo3_rate: float    = Field(default=0., ge=-1., le=1.)

# ── Mod matrix — Amp EG times ─────────────────────────────────────────────────
mod_filter_eg_amp_attack: float  = Field(default=0., ge=-1., le=1.)
mod_filter_eg_amp_decay: float   = Field(default=0., ge=-1., le=1.)
mod_filter_eg_amp_sustain: float = Field(default=0., ge=-1., le=1.)
mod_filter_eg_amp_release: float = Field(default=0., ge=-1., le=1.)
mod_env_1_amp_attack: float      = Field(default=0., ge=-1., le=1.)
mod_env_1_amp_decay: float       = Field(default=0., ge=-1., le=1.)
mod_env_1_amp_sustain: float     = Field(default=0., ge=-1., le=1.)
mod_env_1_amp_release: float     = Field(default=0., ge=-1., le=1.)
mod_env_2_amp_attack: float      = Field(default=0., ge=-1., le=1.)
mod_env_2_amp_decay: float       = Field(default=0., ge=-1., le=1.)
mod_velocity_amp_attack: float   = Field(default=0., ge=-1., le=1.)
mod_velocity_amp_decay: float    = Field(default=0., ge=-1., le=1.)
mod_note_amp_attack: float       = Field(default=0., ge=-1., le=1.)
```

> **Note on schema completeness:** The schema above lists the most musically useful
> source×destination combinations. For full coverage of all 735 routes, generate the
> field list programmatically:
>
> ```python
> SRCS = ["filter_eg","env_1","env_2","env_3","lfo_1","lfo_2","lfo_3",
>         "velocity","note","step","cc1","mpe_pressure","mpe_timbre","mpe_pb","pb"]
> NEW_DSTS = ["flt_key","flt_vel","flt_attack","flt_decay","flt_sustain","flt_release",
>             "lfo1_rate","lfo1_depth","lfo1_phase","lfo1_offset",
>             "lfo2_rate","lfo2_depth","lfo2_phase","lfo2_offset",
>             "lfo3_rate","lfo3_depth","lfo3_phase","lfo3_offset",
>             "amp_attack","amp_decay","amp_sustain","amp_release"]
> for src in SRCS:
>     for dst in NEW_DSTS:
>         print(f'mod_{src}_{dst}: float = Field(default=0., ge=-1., le=1.)')
> ```

---

## 6. Update `app/ai/nl_to_wt_prompt.py` — Add new entries to `WT_DEFAULTS`

In `WT_DEFAULTS`, add zeros for every new mod key. Run this snippet once to generate
the entries, then paste into the defaults dict:

```python
SRCS = ["filter_eg","env_1","env_2","env_3","lfo_1","lfo_2","lfo_3",
        "velocity","note","step","cc1","mpe_pressure","mpe_timbre","mpe_pb","pb"]
NEW_DSTS = ["flt_key","flt_vel","flt_attack","flt_decay","flt_sustain","flt_release",
            "lfo1_rate","lfo1_depth","lfo1_phase","lfo1_offset",
            "lfo2_rate","lfo2_depth","lfo2_phase","lfo2_offset",
            "lfo3_rate","lfo3_depth","lfo3_phase","lfo3_offset",
            "amp_attack","amp_decay","amp_sustain","amp_release"]
for src in SRCS:
    for dst in NEW_DSTS:
        print(f'    "mod_{src}_{dst}": 0.0,')
```

---

## 7. Preset Extractor Update (`scripts/extract_wavetable_presets.py`)

Add the new destination IDs to `MOD_DST_NAME` dict:

```python
MOD_DST_NAME.update({
    "fltkey":    "Filter Key Tracking",
    "fltvel":    "Filter Vel Tracking",
    "fltattack": "Filter EG Attack",
    "fltdecay":  "Filter EG Decay",
    "fltsustain":"Filter EG Sustain",
    "fltrelease":"Filter EG Release",
    "lfo1rate":  "LFO 1 Rate (Hz)",
    "lfo1depth": "LFO 1 Depth",
    "lfo1phase": "LFO 1 Phase",
    "lfo1offset":"LFO 1 Offset",
    "lfo2rate":  "LFO 2 Rate (Hz)",
    "lfo2depth": "LFO 2 Depth",
    "lfo2phase": "LFO 2 Phase",
    "lfo2offset":"LFO 2 Offset",
    "lfo3rate":  "LFO 3 Rate (Hz)",
    "lfo3depth": "LFO 3 Depth",
    "lfo3phase": "LFO 3 Phase",
    "lfo3offset":"LFO 3 Offset",
    "ampattack": "Amp Attack",
    "ampdecay":  "Amp Decay",
    "ampsustain":"Amp Sustain",
    "amprelease":"Amp Release",
})
```

---

## 8. Summary

| File | Change |
|---|---|
| `PluginProcessor.h` | `numDsts` 27 → 49 |
| `PluginProcessor.cpp` | Replace `kModDstIds` / `kModDstNames` arrays (49 entries each) |
| `WavetableVoice.cpp` | Apply modSum for dstIdx 26-48 in filter, LFO, and amp EG sections |
| `wavetable_request.py` | Add new mod_* fields (or generate all 735 programmatically) |
| `nl_to_wt_prompt.py` | Add new entries to `WT_DEFAULTS` |
| `extract_wavetable_presets.py` | Add new destination names to `MOD_DST_NAME` |

After rebuild: **735 mod depth params** (15 sources × 49 destinations).
Every factory preset mod route now has a corresponding explicit VST3 parameter.
