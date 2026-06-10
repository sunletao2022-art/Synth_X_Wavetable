# Full Parameter Exposure — Dev Guide
## Complete Modulation Depth Parameters + Wavetable Loading by Index

**Purpose:** Expose every modulation route and wavetable selection as direct VST3
float/int parameters so Python (and later a transformer encoder) can control the
entire synth state through a single flat parameter vector — no XML injection, no
preset blob manipulation.

---

## 1. Architecture Overview

### What this doc adds

| Feature | Before | After |
|---|---|---|
| Mod routing (env/lfo → param) | XML state blob only | 243 direct float params |
| Wavetable selection | XML state injection hack | 2 direct int params (index 0-214) |
| Total controllable params | 98 | 98 + 243 + 2 = **343** |

### Modulation sources (9)

| ID | Name | Type | Source object in C++ |
|---|---|---|---|
| `feg` | Filter EG | Poly | `filterADSR.getOutput()` |
| `env1` | Envelope 1 | Poly | `modADSRs[0].getOutput()` |
| `env2` | Envelope 2 | Poly | `modADSRs[1].getOutput()` |
| `env3` | Envelope 3 | Poly | `modADSRs[2].getOutput()` |
| `lfo1` | LFO 1 | Poly | `modLFOs[0].getOutput()` |
| `lfo2` | LFO 2 | Poly | `modLFOs[1].getOutput()` |
| `lfo3` | LFO 3 | Poly | `modLFOs[2].getOutput()` |
| `vel` | Velocity | Poly | `currentlyPlayingNote.noteOnVelocity.asUnsignedFloat()` |
| `note` | MIDI Note | Poly | `note.initialNote / 127.0f` |

### Modulation destinations (27)

| Group | Destination ID | Base param | Range note |
|---|---|---|---|
| OSC 1 | `osc1_pos` | `oscParams[0].pos` | 0–100 % |
| OSC 1 | `osc1_tune` | `oscParams[0].tune` | semitones |
| OSC 1 | `osc1_fine` | `oscParams[0].finetune` | cents |
| OSC 1 | `osc1_level` | `oscParams[0].level` | dB |
| OSC 1 | `osc1_pan` | `oscParams[0].pan` | -1–1 |
| OSC 1 | `osc1_detune` | `oscParams[0].detune` | 0–0.5 |
| OSC 1 | `osc1_spread` | `oscParams[0].spread` | -100–100 |
| OSC 1 | `osc1_formant` | `oscParams[0].formant` | -1–1 |
| OSC 1 | `osc1_bend` | `oscParams[0].bend` | -1–1 |
| OSC 2 | `osc2_pos` | `oscParams[1].pos` | 0–100 % |
| OSC 2 | `osc2_tune` | `oscParams[1].tune` | semitones |
| OSC 2 | `osc2_fine` | `oscParams[1].finetune` | cents |
| OSC 2 | `osc2_level` | `oscParams[1].level` | dB |
| OSC 2 | `osc2_pan` | `oscParams[1].pan` | -1–1 |
| OSC 2 | `osc2_detune` | `oscParams[1].detune` | 0–0.5 |
| OSC 2 | `osc2_spread` | `oscParams[1].spread` | -100–100 |
| OSC 2 | `osc2_formant` | `oscParams[1].formant` | -1–1 |
| OSC 2 | `osc2_bend` | `oscParams[1].bend` | -1–1 |
| Sub | `sub_tune` | `subParams.tune` | semitones |
| Sub | `sub_level` | `subParams.level` | dB |
| Sub | `sub_pan` | `subParams.pan` | -1–1 |
| Noise | `noise_level` | `noiseParams.level` | dB |
| Noise | `noise_pan` | `noiseParams.pan` | -1–1 |
| Filter | `flt_freq` | `filterParams.frequency` | MIDI note (0–maxFreq) |
| Filter | `flt_res` | `filterParams.resonance` | 0–100 |
| Filter | `flt_amount` | `filterParams.amount` | -1–1 |
| Master | `master_level` | `globalParams.level` | dB |

**9 sources × 27 destinations = 243 depth parameters.**
All depth params: range `{-1.0, 1.0}`, default `0.0`.
Naming convention: `mod_{src}_{dst}` → e.g. `mod_env1_flt_freq`

### Wavetable index params (2)

| Param ID | Range | Maps to |
|---|---|---|
| `osc1_wt_index` | 0 – 214 (int) | `getWavetableNames()[index]` → OSC 1 |
| `osc2_wt_index` | 0 – 214 (int) | `getWavetableNames()[index]` → OSC 2 |

215 bundled wavetables sorted naturally (same order as `getWavetableNames()`).
Index 0 = "AKWP 0001", index 214 = "NK - WIT". Full list at end of this doc.

---

## 2. C++ Changes

### 2.1 `PluginProcessor.h` — Add `ModDepthParams` struct and `WtIndexParams` struct

Add these two structs inside `WavetableAudioProcessor` after the existing `GlobalParams` struct:

```cpp
//==============================================================================
// Explicit modulation depth parameters.
// Each entry is a VST3 float param (range -1..1, default 0).
// Naming: mod_{source}_{destination}
// Applied additively on top of the gin mod matrix in WavetableVoice::updateParams().
struct ModDepthParams
{
    ModDepthParams() = default;

    // Sources: feg, env1, env2, env3, lfo1, lfo2, lfo3, vel, note
    // Destinations: osc1pos, osc1tune, osc1fine, osc1level, osc1pan,
    //               osc1detune, osc1spread, osc1formant, osc1bend,
    //               osc2pos, osc2tune, osc2fine, osc2level, osc2pan,
    //               osc2detune, osc2spread, osc2formant, osc2bend,
    //               subtune, sublevel, subpan,
    //               noiselevel, noisepan,
    //               fltfreq, fltres, fltamount, masterlevel

    static constexpr int numSrcs  = 9;
    static constexpr int numDsts  = 27;

    // Indexed as depths[srcIdx][dstIdx]
    // srcIdx: 0=feg 1=env1 2=env2 3=env3 4=lfo1 5=lfo2 6=lfo3 7=vel 8=note
    // dstIdx: 0=osc1pos 1=osc1tune 2=osc1fine 3=osc1level 4=osc1pan
    //         5=osc1detune 6=osc1spread 7=osc1formant 8=osc1bend
    //         9=osc2pos 10=osc2tune 11=osc2fine 12=osc2level 13=osc2pan
    //         14=osc2detune 15=osc2spread 16=osc2formant 17=osc2bend
    //         18=subtune 19=sublevel 20=subpan
    //         21=noiselevel 22=noisepan
    //         23=fltfreq 24=fltres 25=fltamount 26=masterlevel
    gin::Parameter::Ptr depths[numSrcs][numDsts];

    void setup (WavetableAudioProcessor& p);

    JUCE_DECLARE_NON_COPYABLE (ModDepthParams)
};

//==============================================================================
// Wavetable index parameters — select bundled wavetable by integer index.
// Index 0..214 maps to getWavetableNames()[index].
struct WtIndexParams
{
    WtIndexParams() = default;

    gin::Parameter::Ptr osc1Index;   // osc1_wt_index
    gin::Parameter::Ptr osc2Index;   // osc2_wt_index

    void setup (WavetableAudioProcessor& p);

    JUCE_DECLARE_NON_COPYABLE (WtIndexParams)
};
```

Also add member variables and a helper method in the `public` section of `WavetableAudioProcessor`:

```cpp
// Modulation depth params
ModDepthParams modDepthParams;

// Wavetable index params
WtIndexParams wtIndexParams;

// Called when osc1_wt_index or osc2_wt_index changes
void setWavetableByIndex (int osc, int index);
```

### 2.2 `PluginProcessor.cpp` — Implement setup methods

Add source and destination name tables, then register all 243 + 2 parameters.

#### Source and destination name tables (add near top of file, in anonymous namespace):

```cpp
namespace
{
    // Source IDs and display names — must match srcIdx order in ModDepthParams
    static constexpr int kNumSrcs = 9;
    static const char* kSrcIds[kNumSrcs]   = { "feg","env1","env2","env3","lfo1","lfo2","lfo3","vel","note" };
    static const char* kSrcNames[kNumSrcs] = { "Filter EG","Env 1","Env 2","Env 3",
                                                "LFO 1","LFO 2","LFO 3","Velocity","Note" };

    // Destination IDs and display names — must match dstIdx order in ModDepthParams
    static constexpr int kNumDsts = 27;
    static const char* kDstIds[kNumDsts] = {
        "osc1pos","osc1tune","osc1fine","osc1level","osc1pan",
        "osc1detune","osc1spread","osc1formant","osc1bend",
        "osc2pos","osc2tune","osc2fine","osc2level","osc2pan",
        "osc2detune","osc2spread","osc2formant","osc2bend",
        "subtune","sublevel","subpan",
        "noiselevel","noisepan",
        "fltfreq","fltres","fltamount","masterlevel"
    };
    static const char* kDstNames[kNumDsts] = {
        "OSC1 Pos","OSC1 Tune","OSC1 Fine","OSC1 Level","OSC1 Pan",
        "OSC1 Detune","OSC1 Spread","OSC1 Formant","OSC1 Bend",
        "OSC2 Pos","OSC2 Tune","OSC2 Fine","OSC2 Level","OSC2 Pan",
        "OSC2 Detune","OSC2 Spread","OSC2 Formant","OSC2 Bend",
        "Sub Tune","Sub Level","Sub Pan",
        "Noise Level","Noise Pan",
        "Flt Freq","Flt Res","Flt Amount","Master Level"
    };
} // namespace
```

#### Implement `ModDepthParams::setup`:

```cpp
void WavetableAudioProcessor::ModDepthParams::setup (WavetableAudioProcessor& p)
{
    for (int s = 0; s < kNumSrcs; ++s)
    {
        for (int d = 0; d < kNumDsts; ++d)
        {
            // Parameter ID:   "mod_feg_osc1pos"
            // Parameter name: "Mod Filter EG OSC1 Pos"
            juce::String id   = juce::String ("mod_") + kSrcIds[s] + "_" + kDstIds[d];
            juce::String name = juce::String ("Mod ") + kSrcNames[s] + " " + kDstNames[d];
            juce::String abbr = juce::String (kSrcIds[s]) + "->" + kDstIds[d];

            depths[s][d] = p.addExtParam (id, name, abbr, "",
                                          { -1.0, 1.0, 0.0, 1.0 },
                                          0.0f,   // default: no modulation
                                          0.0f);
        }
    }
}
```

#### Implement `WtIndexParams::setup`:

```cpp
void WavetableAudioProcessor::WtIndexParams::setup (WavetableAudioProcessor& p)
{
    // 215 wavetables (0-214). Use float range with step 1 — pedalboard exposes as float.
    osc1Index = p.addIntParam ("osc1_wt_index", "OSC1 Wavetable Index", "WT1 Idx", "",
                               { 0.0, 214.0, 1.0, 1.0 }, 0.0f, 0.0f);
    osc2Index = p.addIntParam ("osc2_wt_index", "OSC2 Wavetable Index", "WT2 Idx", "",
                               { 0.0, 214.0, 1.0, 1.0 }, 0.0f, 0.0f);
}
```

#### Call both setup methods in `WavetableAudioProcessor::init()` (or wherever other `setup()` calls are):

```cpp
// Existing calls like:
oscParams[0].setup (*this, 0);
oscParams[1].setup (*this, 1);
filterParams.setup (*this);
// ...

// ADD at the end of the setup block:
modDepthParams.setup (*this);
wtIndexParams.setup  (*this);
```

#### Implement `setWavetableByIndex`:

```cpp
void WavetableAudioProcessor::setWavetableByIndex (int osc, int index)
{
    auto names = getWavetableNames();
    if (index < 0 || index >= names.size())
        return;

    if (osc == 0)
    {
        userTable1.reset();
        osc1Table = names[index];
    }
    else
    {
        userTable2.reset();
        osc2Table = names[index];
    }

    reloadWavetables();
}
```

#### Hook `WtIndexParams` into the parameter change callback

In `WavetableAudioProcessor`, override `gin::Processor::parameterChanged` (or add a
`juce::AudioProcessorValueTreeState::Listener`). The cleanest gin approach is to add
a `ChangeListener` on the parameter object. Find the existing `stateUpdated()` method
and add at the end:

```cpp
void WavetableAudioProcessor::stateUpdated (const juce::ValueTree& v)
{
    // ... existing code ...
    modMatrix.stateUpdated (v);
    reloadWavetables();

    // Sync wt index params → actual table names after state load
    auto names = getWavetableNames();
    int idx1 = int (wtIndexParams.osc1Index->getUserValue());
    int idx2 = int (wtIndexParams.osc2Index->getUserValue());
    if (idx1 >= 0 && idx1 < names.size()) { userTable1.reset(); osc1Table = names[idx1]; }
    if (idx2 >= 0 && idx2 < names.size()) { userTable2.reset(); osc2Table = names[idx2]; }
    reloadWavetables();
}
```

And override `parameterChanged` to respond to live index changes during playback:

```cpp
void WavetableAudioProcessor::parameterChanged (gin::Parameter* param, float)
{
    if (param == wtIndexParams.osc1Index)
        setWavetableByIndex (0, int (wtIndexParams.osc1Index->getUserValue()));
    else if (param == wtIndexParams.osc2Index)
        setWavetableByIndex (1, int (wtIndexParams.osc2Index->getUserValue()));
}
```

Register the listener after the params are set up:

```cpp
// In init(), after wtIndexParams.setup(*this):
wtIndexParams.osc1Index->addListener (this);
wtIndexParams.osc2Index->addListener (this);
```

`WavetableAudioProcessor` needs to inherit from `gin::Parameter::ParameterListener`
(or use a lambda listener — check gin's API for the preferred pattern in this codebase).

### 2.3 `WavetableVoice.cpp` — Apply modulation depths in `updateParams()`

At the top of `updateParams()`, after the existing modulator outputs are computed,
collect all source values into a local array. Then apply them per-destination.

#### Step A — Collect source outputs

Add this block **after** all the existing modulator `.process(blockSize)` calls
(filter ADSR, modADSRs, modLFOs) but **before** the existing `filter.setParams()` call:

```cpp
// ── Collect modulation source outputs ────────────────────────────────────────
// srcIdx: 0=feg 1=env1 2=env2 3=env3 4=lfo1 5=lfo2 6=lfo3 7=vel 8=note
float modSrcOut[9];
modSrcOut[0] = filterADSR.getOutput();
modSrcOut[1] = proc.envParams[0].enable->isOn() ? modADSRs[0].getOutput() : 0.0f;
modSrcOut[2] = proc.envParams[1].enable->isOn() ? modADSRs[1].getOutput() : 0.0f;
modSrcOut[3] = proc.envParams[2].enable->isOn() ? modADSRs[2].getOutput() : 0.0f;
modSrcOut[4] = proc.lfoParams[0].enable->isOn() ? modLFOs[0].getOutput() : 0.0f;
modSrcOut[5] = proc.lfoParams[1].enable->isOn() ? modLFOs[1].getOutput() : 0.0f;
modSrcOut[6] = proc.lfoParams[2].enable->isOn() ? modLFOs[2].getOutput() : 0.0f;
modSrcOut[7] = currentlyPlayingNote.noteOnVelocity.asUnsignedFloat();
modSrcOut[8] = currentlyPlayingNote.initialNote / 127.0f;

// Helper: sum all source contributions for one destination slot
auto modSum = [&] (int dstIdx) -> float
{
    float total = 0.0f;
    for (int s = 0; s < 9; ++s)
        total += modSrcOut[s] * proc.modDepthParams.depths[s][dstIdx]->getUserValue();
    return total;
};
```

#### Step B — Apply to OSC params

Replace / extend the existing oscillator parameter reads in `updateParams()`:

```cpp
// OSC 1  (dstIdx 0-8)
oscParams[0].position = juce::jlimit (0.0f, 1.0f,
    (getValue (proc.oscParams[0].pos) + modSum(0) * 100.0f) / 100.0f);

// Pitch is applied to currentMidiNotes, add mod contribution in semitones
currentMidiNotes[0] += modSum(1) * 36.0f;   // dstIdx 1 = osc1tune (±36 st range)
currentMidiNotes[0] += modSum(2) * 100.0f / 100.0f;  // dstIdx 2 = osc1fine (±100 ct)

oscParams[0].gain   = juce::jlimit (0.0f, 1.0f,
    gin::velocityToGain (currentlyPlayingNote.noteOnVelocity.asUnsignedFloat(), ampKeyTrack)
    * std::pow (10.0f, (getValue (proc.oscParams[0].level) + modSum(3) * 100.0f) / 20.0f));
    // Note: simplification — real dB→linear calc already done elsewhere, adjust accordingly

oscParams[0].pan     = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[0].pan)    + modSum(4));
oscParams[0].detune  = juce::jlimit ( 0.0f, 0.5f, getValue (proc.oscParams[0].detune) + modSum(5) * 0.5f);
oscParams[0].spread  = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[0].spread) / 100.0f + modSum(6));
oscParams[0].formant = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[0].formant)+ modSum(7));
oscParams[0].bend    = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[0].bend)   + modSum(8));

// OSC 2  (dstIdx 9-17) — same pattern, shift dstIdx by 9
oscParams[1].position = juce::jlimit (0.0f, 1.0f,
    (getValue (proc.oscParams[1].pos) + modSum(9) * 100.0f) / 100.0f);
currentMidiNotes[1] += modSum(10) * 36.0f;
currentMidiNotes[1] += modSum(11);
oscParams[1].pan     = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[1].pan)    + modSum(13));
oscParams[1].detune  = juce::jlimit ( 0.0f, 0.5f, getValue (proc.oscParams[1].detune) + modSum(14) * 0.5f);
oscParams[1].spread  = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[1].spread) / 100.0f + modSum(15));
oscParams[1].formant = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[1].formant)+ modSum(16));
oscParams[1].bend    = juce::jlimit (-1.0f, 1.0f, getValue (proc.oscParams[1].bend)   + modSum(17));

// Sub  (dstIdx 18-20)
subNote += modSum(18) * 36.0f;
subParams.leftGain  *= juce::jlimit (0.0f, 2.0f, 1.0f + modSum(19));
subParams.rightGain *= juce::jlimit (0.0f, 2.0f, 1.0f + modSum(19));
subParams.leftGain  = juce::jlimit (-1.0f, 1.0f, subParams.leftGain  * (1.0f - modSum(20)));
subParams.rightGain = juce::jlimit (-1.0f, 1.0f, subParams.rightGain * (1.0f + modSum(20)));

// Noise (dstIdx 21-22)
noiseParams.leftGain  *= juce::jlimit (0.0f, 2.0f, 1.0f + modSum(21));
noiseParams.rightGain *= juce::jlimit (0.0f, 2.0f, 1.0f + modSum(21));
```

#### Step C — Apply to filter (dstIdx 23-25)

Extend the existing filter calculation block:

```cpp
// Existing:
float n = getValue (proc.filterParams.frequency);
n += (currentlyPlayingNote.initialNote - 60) * getValue (proc.filterParams.keyTracking);
n += filterEnv * filterSens * getValue (proc.filterParams.amount) * filterWidth;

// ADD: explicit mod depth contributions
n += modSum(23) * filterWidth;    // dstIdx 23 = flt_freq (full pitch range)

float resBase  = getValue (proc.filterParams.resonance);
resBase = juce::jlimit (0.0f, 100.0f, resBase + modSum(24) * 100.0f);  // dstIdx 24 = flt_res

// flt_amount itself can be modulated (dstIdx 25) — useful for dynamic EG depth
float amtBase  = getValue (proc.filterParams.amount);
amtBase = juce::jlimit (-1.0f, 1.0f, amtBase + modSum(25));
// Re-apply EG→cutoff with modulated amount
n += filterEnv * filterSens * amtBase * filterWidth;

float f = gin::getMidiNoteInHertz (n);
float maxFreq = std::min (20000.0f, float (getSampleRate() / 2));
f = juce::jlimit (4.0f, maxFreq, f);
float q = gin::Q / (1.0f - (resBase / 100.0f) * 0.99f);
filter.setParams (f, q);
```

#### Step D — Apply to master level (dstIdx 26)

This is a global (mono) parameter — apply in `PluginProcessor.cpp`'s block-level
processing rather than in the voice, since master level is not per-voice:

```cpp
// In processBlock(), after all voice rendering, before FX:
// Compute modSum for master level across active voices (use average or max)
// For simplicity, read from a designated voice or use mono mod sources.
// Simplest approach: expose as a direct multiplier applied post-mix.
// (Implementation detail: tie to the existing `globalParams.level` path.)
```

> **Note for transformer training:** master level modulation is rarely used in
> practice. If it causes implementation complexity, skip dstIdx 26 initially —
> the transformer will simply learn to output 0 for it.

---

## 3. Wavetable Index Reference

215 bundled wavetables, index 0–214, sorted naturally (same order as `getWavetableNames()`):

```
Index  Name
0      AKWP 0001
1      AKWP 0002
...    (AKWP 0003–0020: indices 2–19)
20     AKWP aguitar
21     AKWP altosax
22     AKWP birds
23     AKWP bitreduced
24     AKWP bw_blended
25     AKWP bw_perfectwaves
26     AKWP bw_saw
27     AKWP bw_sawbright
28     AKWP bw_sawgap
29     AKWP bw_sawrounded
30     AKWP bw_sin
31     AKWP bw_squ
32     AKWP bw_squrounded
33     AKWP bw_tri
34     AKWP c604
35     AKWP cello
36     AKWP clarinett
37     AKWP clavinet
38     AKWP dbass
39     AKWP distorted
40     AKWP ebass
41     AKWP eguitar
42     AKWP eorgan
43     AKWP epiano
44     AKWP flute
45     AKWP fmsynth
46     AKWP granular
47     AKWP hdrawn
48     AKWP hvoice
49     AKWP linear
50     AKWP oboe
51     AKWP oscchip
52     AKWP overtone
53     AKWP piano
54     AKWP pluckalgo
55     AKWP raw
56     AKWP sinharm
57     AKWP snippets
58     AKWP stereo
59     AKWP stringbox
60     AKWP symetric
61     AKWP theremin
62     AKWP vgame
63     AKWP vgamebasic
64     AKWP violin
65     Analog Classic
66     Analog PWM Custom 01
...    (Custom 02–20: indices 67–85)
86     Analog PWM Saw 01
...    (Saw 02–07: indices 87–92)
93     Analog PWM Square 01
...    (Square 02–06: indices 94–98)
99     Analog PWM Sub 01
...    (Sub 02–08: indices 100–106)
107    Dist - Asym
108    Dist - Diode 1
109    Dist - Diode 2
110    Dist - Downsample
111    Dist - Filter Drive
112    Dist - Hard Clipping
113    Dist - Hard Clipping Extra
114    Dist - Lin.Fold
115    Dist - Rectify
116    Dist - Sin.Fold
117    Dist - Sine Shaper
118    Dist - Soft Clipping
119    Dist - Stomp Box
120    Dist - Tape
121    Dist - Tube
122    Dist - Zero Square
123    FM 01
...    (FM 02–17: indices 124–139)
140    Growl 01
...    (Growl 02–18: indices 141–157)
158    Hyper 01
...    (Hyper 02–07: indices 159–164)
165    NK - ACTIVE
...    (NK series: indices 165–214)
214    NK - WIT
```

To get the exact index for any wavetable name from Python:

```python
from pedalboard import load_plugin
p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
# Read osc1_wt_index range to confirm 215 tables
print(p.parameters['osc1_wt_index'])
```

Or generate the full index map:

```python
# Run once after the C++ changes are built
# The plugin exposes osc1_wt_index as a param; the names are implicit in the index.
# Build a lookup dict from the known sorted list:
import json, pathlib

wt_dir = pathlib.Path('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3/Contents/Resources/Wavetables')
names = sorted([f.stem for f in wt_dir.rglob('*.wt2048')])  # natural sort may differ — use plugin order
wt_index = {name: idx for idx, name in enumerate(names)}
print(json.dumps(wt_index, indent=2))
```

---

## 4. Build Commands

```bash
cd /path/to/Synth_X/Wavetable

# Debug build (fast iteration)
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --config Debug --target Wavetable_VST3 -j$(sysctl -n hw.logicalcpu)

# Install (user folder, no sudo)
cp -r build/Wavetable_artefacts/Debug/VST3/Wavetable.vst3 \
      ~/Library/Audio/Plug-Ins/VST3/Wavetable.vst3

# Verify param count
python3 -c "
from pedalboard import load_plugin
p = load_plugin('$HOME/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
print('Total params:', len(p.parameters))
mod_params = [k for k in p.parameters if k.startswith('mod_')]
print('Mod depth params:', len(mod_params))
wt_params = [k for k in p.parameters if 'wt_index' in k]
print('Wavetable index params:', wt_params)
"
# Expected: Total params: 343, Mod depth params: 243, Wavetable index params: ['osc1_wt_index', 'osc2_wt_index']
```

---

## 5. Python Interface

### Set a modulation route

```python
from pedalboard import load_plugin
p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')

# Route env1 → filter cutoff with depth 0.8
p.mod_env1_fltfreq = 0.8

# Route lfo1 → osc1 wavetable position with depth 0.5
p.mod_lfo1_osc1pos = 0.5

# Route velocity → filter resonance with depth -0.4 (high velocity = less resonance)
p.mod_vel_fltres = -0.4

# Route filter EG → osc1 tune with depth 0.2 (EG pushes pitch up slightly)
p.mod_feg_osc1tune = 0.2
```

### Load a wavetable by index

```python
# Load "Analog Classic" (index 65) on OSC 1
p.osc1_wt_index = 65

# Load "AKWP bw_saw" (index 26) on OSC 2
p.osc2_wt_index = 26

# Load by name (requires the lookup dict)
wt_index = { "Analog Classic": 65, "AKWP bw_saw": 26, ... }
p.osc1_wt_index = wt_index["Analog Classic"]
```

### Reset all modulation to zero

```python
for key in p.parameters:
    if key.startswith('mod_'):
        setattr(p, key, 0.0)
```

### Full patch dict for transformer output

```python
import numpy as np

def params_to_dict(plugin):
    return {k: float(getattr(plugin, k)) for k in plugin.parameters}

def apply_param_vector(plugin, vector, param_names):
    """Apply a flat numpy array (transformer output) to the plugin."""
    for name, value in zip(param_names, vector):
        try:
            setattr(plugin, name, float(value))
        except Exception:
            pass

# Get ordered param name list (stable across calls)
param_names = list(p.parameters.keys())   # length 343 after changes
# Transformer output shape: (343,)
```

---

## 6. Transformer Training Notes

### Parameter vector structure (343 floats)

```
[0:98]    Base parameters (existing — oscillators, filter, LFOs, FX, etc.)
[98:341]  Mod depth matrix: depths[src][dst] row-major, src=0..8, dst=0..26
[341]     osc1_wt_index  (0–214, integer semantics but float in VST3)
[342]     osc2_wt_index  (0–214, integer semantics but float in VST3)
```

### Why unused routes are not a problem

Routes the training data never uses will consistently have ground-truth label = 0.
The output head learns to predict 0 for those slots trivially. MSE loss contribution
from always-zero outputs is negligible after the first few epochs.

### Wavetable index as a training target

`osc1_wt_index` is a categorical variable (215 classes) embedded as an integer float.
For the transformer output you have two options:

1. **Regression** — output a single float, round to nearest int at inference. Simple,
   works if the model can learn the integer quantization. Loss: `MSE(pred, true_idx / 214)`.
2. **Classification head** — separate 215-class softmax head for each oscillator,
   take argmax. More principled, better calibrated, slightly more complex architecture.

Recommendation: start with regression (it keeps the output head uniform), switch to
a classification head if the model struggles with wavetable selection accuracy.

### Normalization

All mod depth params are already in `[-1, 1]` — no normalization needed.
Base params have varying ranges — normalize each to `[0, 1]` using the `min`/`max`
from `p.parameters[key]` range before training.
Wavetable index: normalize to `[0, 1]` as `index / 214`.

### Data generation tip

When generating training data, first zero all mod depths, then selectively enable
routes that match the archetype being generated. This creates a sparse but
structured dataset — the transformer learns sparsity naturally and won't hallucinate
modulation that shouldn't be there.
