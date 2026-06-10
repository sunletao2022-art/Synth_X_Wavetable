# Filter Envelope Modulation — Development Guide

**Goal:** Add Serum/Surge-style envelope-to-filter-parameter routing so the filter EG
can directly modulate **resonance** (and optionally drive/keytracking depth) via a
dedicated amount knob, on top of the existing EG→cutoff path.

---

## 1. Architecture Audit — What Already Works

Before writing any code, understand what the plugin already has.

### The `gin::ModMatrix` is fully wired

`PluginProcessor.cpp` registers every non-internal parameter as a mod destination:

```cpp
// PluginProcessor.cpp ~line 861
for (auto pp : getPluginParameters())
{
    if (pp == firstMonoParam)
        polyParam = false;
    if (!pp->isInternal() || pp == delayParams.delay)
        modMatrix.addParameter(pp, polyParam, getSmoothingTime(pp));
}
modMatrix.build();
```

This means `filterParams.frequency`, `filterParams.resonance`, and `filterParams.amount`
are **already registered mod destinations**. Any mod source (LFO, Env 1-3, velocity,
CC, MPE) can already be routed to resonance via the mod matrix UI.

### The Filter EG output is already a mod source

```cpp
// WavetableVoice.cpp
proc.modMatrix.setPolyValue(*this, proc.modSrcFilter, filterADSR.getOutput());
```

So "Filter Envelope" appears as a source in the mod matrix and can be dragged to
any destination including resonance.

### `getValue()` reads through the mod matrix

In `WavetableVoice::updateParams()`:
```cpp
float q = gin::Q / (1.0f - (getValue(proc.filterParams.resonance) / 100.0f) * 0.99f);
```

`getValue(param)` in the gin framework returns `base_value + sum_of_modulations`.
So routing Filter Env → Resonance in the mod matrix UI **already works at DSP level**
with no code changes.

### What is hardcoded (the gap)

The EG→cutoff path bypasses the mod matrix:

```cpp
// WavetableVoice.cpp ~line 337
float n = getValue(proc.filterParams.frequency);          // mod-matrix-aware ✓
n += filterEnv * filterSens                               // HARDCODED ← this
   * getValue(proc.filterParams.amount)
   * filterWidth;
```

There is no parallel hardcoded path for resonance. The `amount` knob only drives
cutoff. That is the only thing missing.

---

## 2. Surge XT Reference

Surge XT uses the same two-tier pattern:

- **Hardcoded shortcut** — Filter 1 Cutoff has a dedicated `f_eg_gain` slot that
  directly scales EG2's output onto cutoff, controlled by a signed knob in the
  filter panel. This is equivalent to Wavetable's existing `amount` knob.
- **Mod matrix** — Everything else (resonance, drive, keytrack depth, second filter
  cutoff, etc.) is routed via the mod matrix. EG1, EG2, and all LFOs are sources;
  every parameter is a destination. Resonance modulation is done by adding a route
  `EG2 → Filter Resonance` with the desired depth.

See `surge/src/common/dsp/SurgeVoice.cpp` — search for `fbval` and `coeff_LP` for
the hardcoded cutoff path, then look at how `modsources[ms_filtereg]` feeds into
the mod routing system.

**Takeaway for Wavetable:** implement a dedicated `resAmount` knob (the hardcoded
shortcut) so users get a single visible knob in the filter section, exactly like
Surge's cutoff amount knob. Users who want deeper routing (LFO → resonance, Env 2
→ resonance) already have it via the mod matrix.

---

## 3. Changes Required

### 3.1 `PluginProcessor.h` — Add `resAmount` to `FilterParams`

```cpp
// Before:
struct FilterParams
{
    gin::Parameter::Ptr enable, type, keyTracking, velocityTracking,
                        frequency, resonance, amount, retrig,
                        attack, decay, sustain, release, wt1, wt2, sub, noise;
    ...
};

// After: add resAmount
struct FilterParams
{
    gin::Parameter::Ptr enable, type, keyTracking, velocityTracking,
                        frequency, resonance, amount, resAmount, retrig,
                        attack, decay, sustain, release, wt1, wt2, sub, noise;
    ...
};
```

### 3.2 `PluginProcessor.cpp` — Register the new parameter

In `FilterParams::setup()`, after the `amount` line (~line 179):

```cpp
// Existing:
amount    = p.addExtParam(id + "amount",    nm + "Amount",    "Amnt",    "",  { -1.0, 1.0, 0.0, 1.0 }, 0.0,  0.0f);

// Add directly below:
resAmount = p.addExtParam(id + "resAmount", nm + "Res Amnt",  "ResAmnt", "",  { -1.0, 1.0, 0.0, 1.0 }, 0.0,  0.0f);
```

The `id` prefix ensures a unique parameter ID (e.g. `"fltresAmount"`). The range
`{-1, 1}` matches the existing `amount` knob so the EG can push resonance up or down.

### 3.3 `WavetableVoice.cpp` — Wire `resAmount` into the DSP

In `WavetableVoice::updateParams()`, directly after the resonance `q` calculation:

```cpp
// Existing block (around line 337–345):
float n = getValue(proc.filterParams.frequency);
n += (currentlyPlayingNote.initialNote - 60) * getValue(proc.filterParams.keyTracking);
n += filterEnv * filterSens * getValue(proc.filterParams.amount) * filterWidth;

float f = gin::getMidiNoteInHertz(n);
float maxFreq = std::min(20000.0f, float(getSampleRate() / 2));
f = juce::jlimit(4.0f, maxFreq, f);

float q = gin::Q / (1.0f - (getValue(proc.filterParams.resonance) / 100.0f) * 0.99f);

// ADD: filter EG → resonance (hardcoded shortcut, mirrors the cutoff path)
float resBase   = getValue(proc.filterParams.resonance);          // already mod-matrix modulated
float resEGMod  = filterEnv * filterSens
                * getValue(proc.filterParams.resAmount)           // new knob
                * 100.0f;                                         // scale to 0-100 range
float resFinal  = juce::jlimit(0.0f, 100.0f, resBase + resEGMod);
q = gin::Q / (1.0f - (resFinal / 100.0f) * 0.99f);

filter.setParams(f, q);
```

**Note:** `resBase` already includes any mod-matrix modulations (LFO → resonance,
Env 1 → resonance, etc.) because `getValue()` sums them. The `resEGMod` line then
adds the hardcoded filter-EG shortcut on top. This mirrors exactly how Surge XT
layers its hardcoded EG→cutoff path over the mod matrix output.

### 3.4 `Panels.h` — EG Depth subgroup with Cutoff / Res labels

Instead of adding a bare knob, replace the single `amount` knob with a labelled
two-knob subgroup. The subgroup sits in the same grid slot as the old `amount`
knob and shows both targets side by side with text labels underneath, identical
to how Serum labels its ENV depth section.

#### Step A — Define `FilterEGDepthBox` above `FilterBox`

Add this class immediately before the `class FilterBox` definition in `Panels.h`:

```cpp
//==============================================================================
// Two-knob subgroup: Filter EG depth for Cutoff and Resonance.
// Displayed as a labelled pair, replacing the old single "Amount" knob.
class FilterEGDepthBox : public juce::Component
{
public:
    FilterEGDepthBox (WavetableAudioProcessor& proc)
    {
        cutoffKnob = std::make_unique<gin::Knob> (proc.filterParams.amount,   true);
        resKnob    = std::make_unique<gin::Knob> (proc.filterParams.resAmount, true);

        addAndMakeVisible (*cutoffKnob);
        addAndMakeVisible (*resKnob);
    }

    void resized() override
    {
        // Split width evenly between the two knobs
        auto area  = getLocalBounds();
        int  half  = area.getWidth() / 2;
        int  knobH = area.getHeight() - 14;  // leave 14px for label text

        cutoffKnob->setBounds (area.removeFromLeft (half).withHeight (knobH));
        resKnob   ->setBounds (area.withHeight (knobH));
    }

    void paint (juce::Graphics& g) override
    {
        auto area   = getLocalBounds();
        int  half   = area.getWidth() / 2;
        int  labelY = getHeight() - 14;

        g.setFont (juce::Font (10.0f));
        g.setColour (findColour (gin::PluginLookAndFeel::whiteColourId).withAlpha (0.6f));

        // "Cutoff" label centred under left knob
        g.drawText ("Cutoff", 0,     labelY, half,        14, juce::Justification::centred);
        // "Res" label centred under right knob
        g.drawText ("Res",    half,  labelY, area.getWidth() - half, 14, juce::Justification::centred);
    }

    gin::Knob* getCutoffKnob() { return cutoffKnob.get(); }
    gin::Knob* getResKnob()    { return resKnob.get(); }

private:
    std::unique_ptr<gin::Knob> cutoffKnob;
    std::unique_ptr<gin::Knob> resKnob;
};
```

#### Step B — Update `FilterBox` to use `FilterEGDepthBox`

Inside `FilterBox`, replace the old `amount` knob with the new subgroup. Add a
member pointer at the top of the class alongside the existing `v`, `a`, `d`, etc.:

```cpp
// Add to FilterBox private members:
FilterEGDepthBox* egDepth = nullptr;
```

In the constructor, replace:

```cpp
// REMOVE this line:
addControl(new gin::Knob(flt.amount, true));
```

with:

```cpp
// ADD: labelled EG depth subgroup (Cutoff + Res)
egDepth = new FilterEGDepthBox (proc);
addControl (egDepth);   // occupies the same grid slot as the old amount knob
```

The `addControl` without explicit grid coordinates appends to the flow layout, so
it naturally lands in the same position as the removed `amount` knob.

#### Step C — Update `paramChanged()` and `watchParam`

```cpp
void paramChanged() override
{
    gin::ParamBox::paramChanged();
    auto& flt = proc.filterParams;

    bool egActive = flt.amount->getUserValue()    != 0.0f
                 || flt.resAmount->getUserValue()  != 0.0f;

    v    ->setEnabled (egActive);
    a    ->setEnabled (egActive);
    d    ->setEnabled (egActive);
    s    ->setEnabled (egActive);
    r    ->setEnabled (egActive);
    adsr ->setEnabled (egActive);

    if (retrig != nullptr)
        retrig->setVisible (proc.globalParams.mono->isOn()
                         && proc.globalParams.glideMode->getUserValue() > 0);
}
```

Add both watch calls so `paramChanged` fires when either knob moves:

```cpp
watchParam (flt.amount);
watchParam (flt.resAmount);   // ← add this
```

#### What it looks like

```
┌─────────────────────────────────────┐
│  Freq   Res   [Cutoff | Res]  Key   │  ← filter top row
│               └── EG Depth ──┘      │
│  Type   Vel   [ADSR display  ]      │  ← filter bottom row
│               A    D    S    R      │
└─────────────────────────────────────┘
```

The `[Cutoff | Res]` subgroup replaces the old single `Amount` knob. Both knobs
are bipolar (centre = 0, up = positive EG push, down = negative). Labels "Cutoff"
and "Res" sit underneath each knob in smaller text.

---

## 4. Build Commands

```bash
cd /path/to/Synth_X/Wavetable

# First-time configure (Debug build for fast iteration)
cmake -B build -DCMAKE_BUILD_TYPE=Debug

# Build the VST3
cmake --build build --config Debug --target Wavetable_VST3 -j$(sysctl -n hw.logicalcpu)

# Install to system VST3 folder (requires sudo on macOS)
sudo cp -r build/Wavetable_artefacts/Debug/VST3/Wavetable.vst3 \
           /Library/Audio/Plug-Ins/VST3/Wavetable.vst3

# Or install to user folder (no sudo needed)
cp -r build/Wavetable_artefacts/Debug/VST3/Wavetable.vst3 \
      ~/Library/Audio/Plug-Ins/VST3/Wavetable.vst3
```

For a Release build (smaller binary, better performance):

```bash
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release --target Wavetable_VST3 -j$(sysctl -n hw.logicalcpu)
sudo cp -r build_release/Wavetable_artefacts/Release/VST3/Wavetable.vst3 \
           /Library/Audio/Plug-Ins/VST3/Wavetable.vst3
```

Verify the new parameter is exposed after install:

```bash
cd /path/to/Synth_X/AI_Synth
python3 -c "
from pedalboard import load_plugin
p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
params = [k for k in p.parameters if 'res' in k.lower()]
print('Resonance-related params:', params)
"
```

You should see `fltresAmount` (or similar) appear in the list.

---

## 5. Test Commands

### 5.1 Parameter existence check

```bash
python3 -c "
from pedalboard import load_plugin
p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
for k in sorted(p.parameters):
    if 'flt' in k.lower():
        print(k, '=', getattr(p, k))
"
```

Expected output includes `fltresAmount = 0.0`.

### 5.2 Static resonance sanity check

```bash
cd /path/to/Synth_X/AI_Synth
python3 -c "
from pedalboard import load_plugin
import numpy as np, wave

p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
SR = 44100

# Baseline: high resonance, no EG modulation
p.fltEnable     = 1.0
p.fltFreq       = 800.0
p.fltRes        = 80.0      # high resonance
p.fltAmount     = 0.0       # EG→cutoff off
p.fltresAmount  = 0.0       # EG→resonance off
p.fltAttack     = 0.01
p.fltDecay      = 0.3
p.fltSustain    = 0.0
p.fltRelease    = 0.1

audio = p([], duration=1.5, sample_rate=SR, num_channels=2,
           midi_messages=[([0x90, 60, 100], 0.0), ([0x80, 60, 0], 1.0)])
peak_baseline = float(np.max(np.abs(audio)))
print(f'Baseline peak (res=80, resAmnt=0): {peak_baseline:.4f}')

# With resAmount = -1.0: EG should pull resonance DOWN (less ringing)
p.fltresAmount = -1.0
audio2 = p([], duration=1.5, sample_rate=SR, num_channels=2,
            midi_messages=[([0x90, 60, 100], 0.0), ([0x80, 60, 0], 1.0)])
peak_low = float(np.max(np.abs(audio2)))
print(f'Peak with resAmnt=-1.0 (EG damps resonance): {peak_low:.4f}')

# With resAmount = +1.0: EG should push resonance UP (more self-oscillation)
p.fltresAmount = 1.0
audio3 = p([], duration=1.5, sample_rate=SR, num_channels=2,
            midi_messages=[([0x90, 60, 100], 0.0), ([0x80, 60, 0], 1.0)])
peak_high = float(np.max(np.abs(audio3)))
print(f'Peak with resAmnt=+1.0 (EG boosts resonance): {peak_high:.4f}')

# Sanity: -1 should reduce peak vs baseline, +1 should increase it
assert peak_low < peak_baseline, 'FAIL: negative resAmount did not reduce resonance'
assert peak_high > peak_baseline, 'FAIL: positive resAmount did not boost resonance'
print('PASS: resAmount modulates resonance correctly')
"
```

### 5.3 Render WAV comparison test

```bash
python3 -c "
from pedalboard import load_plugin, Pedalboard
import numpy as np, wave, struct

def render(plugin, note=60, velocity=100, duration=2.0, sr=44100):
    midi = [([0x90, note, velocity], 0.0), ([0x80, note, 0], duration - 0.1)]
    audio = plugin([], duration=duration, sample_rate=sr, num_channels=2, midi_messages=midi)
    return np.asarray(audio, dtype=np.float32)

def save_wav(path, audio, sr=44100):
    if audio.ndim == 2 and audio.shape[0] == 2:
        audio = audio.T
    pcm = (audio * 32767).clip(-32767, 32767).astype(np.int16)
    with wave.open(path, 'w') as f:
        f.setnchannels(2); f.setsampwidth(2); f.setframerate(sr)
        f.writeframes(pcm.tobytes())

p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
p.fltEnable = 1.0; p.fltFreq = 600.0; p.fltRes = 60.0
p.fltAttack = 0.001; p.fltDecay = 0.4; p.fltSustain = 0.0; p.fltRelease = 0.1

p.fltAmount = 0.0; p.fltresAmount = 0.0
save_wav('/tmp/filter_baseline.wav', render(p))
print('Saved: /tmp/filter_baseline.wav  (no EG mod)')

p.fltAmount = 0.5; p.fltresAmount = 0.0
save_wav('/tmp/filter_cutoff_eg.wav', render(p))
print('Saved: /tmp/filter_cutoff_eg.wav  (EG→cutoff only)')

p.fltAmount = 0.0; p.fltresAmount = 0.8
save_wav('/tmp/filter_res_eg.wav', render(p))
print('Saved: /tmp/filter_res_eg.wav    (EG→resonance only)')

p.fltAmount = 0.5; p.fltresAmount = 0.8
save_wav('/tmp/filter_both_eg.wav', render(p))
print('Saved: /tmp/filter_both_eg.wav   (EG→cutoff + resonance)')
"
```

Open the four WAVs in any DAW or use `open /tmp/filter_*.wav` on macOS. You should
hear clear timbral differences between them, especially the resonance-only file which
should sound like the filter's Q-peak blooms and decays without cutoff moving.

### 5.4 Run existing wavetable_service tests

```bash
cd /path/to/Synth_X/AI_Synth
python3 -m pytest tests/test_filter_changes.py -v
```

---

## 6. Summary of Changes

| File | What changes |
|---|---|
| `plugin/Source/PluginProcessor.h` | Add `resAmount` to `FilterParams` struct |
| `plugin/Source/PluginProcessor.cpp` | Register `resAmount` via `addExtParam` in `FilterParams::setup()` |
| `plugin/Source/WavetableVoice.cpp` | Apply hardcoded EG→resonance path using `filterEnv * getValue(resAmount)` |
| `plugin/Source/Panels.h` | Add knob to `FilterBox`, update `paramChanged()` and `watchParam` |

No changes needed to the mod matrix wiring — resonance modulation via LFOs and
additional envelopes already works through the existing `getValue()` / mod matrix
path. This change only adds the dedicated per-note filter-EG shortcut knob.
