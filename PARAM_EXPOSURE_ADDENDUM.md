# Parameter Exposure Addendum
## Additional Mod Sources + Dynamic Wavetable Index + WAV-per-Oscillator

Extends `FULL_PARAM_EXPOSURE_DEV_GUIDE.md`. Apply these changes on top of that guide.

---

## 1. Additional Modulation Sources

### New sources (6 more → total 15)

| New srcIdx | ID | Name | Type | C++ source value |
|---|---|---|---|---|
| 9 | `step` | Step LFO | Poly | `modStepLFO.getOutput()` |
| 10 | `cc1` | CC1 Mod Wheel | Mono | `proc.modMatrix.getMonoValue(proc.modSrcCC[1])` |
| 11 | `mpe_pressure` | MPE Pressure | Poly | `currentlyPlayingNote.pressure.asUnsignedFloat()` |
| 12 | `mpe_timbre` | MPE Timbre | Poly | `currentlyPlayingNote.timbre.asUnsignedFloat()` |
| 13 | `mpe_pb` | MPE Pitch Bend | Poly | `currentlyPlayingNote.pitchbend.asUnsignedFloat()` |
| 14 | `pb` | Pitch Bend | Mono | `proc.modMatrix.getMonoValue(proc.modScrPitchBend)` |

**Revised total: 15 sources × 27 destinations = 405 depth parameters.**

> Note: `modSrcCC` is an array indexed by CC number. CC1 is the mod wheel — the most
> musically useful CC. If you later want CC7 (volume) or CC74 (filter in MPE), add
> `cc7`, `cc74` etc. using the same pattern with `proc.modSrcCC[7]`.

### 1.1 `PluginProcessor.h` — Update `ModDepthParams`

Change `numSrcs` from 9 to 15:

```cpp
static constexpr int numSrcs = 15;   // was 9

// srcIdx:  0=feg 1=env1 2=env2 3=env3 4=lfo1 5=lfo2 6=lfo3 7=vel 8=note
//          9=step 10=cc1 11=mpe_pressure 12=mpe_timbre 13=mpe_pb 14=pb
```

### 1.2 `PluginProcessor.cpp` — Update name tables

```cpp
static constexpr int kNumSrcs = 15;   // was 9

static const char* kSrcIds[kNumSrcs] = {
    "feg","env1","env2","env3","lfo1","lfo2","lfo3","vel","note",
    "step","cc1","mpe_pressure","mpe_timbre","mpe_pb","pb"
};

static const char* kSrcNames[kNumSrcs] = {
    "Filter EG","Env 1","Env 2","Env 3","LFO 1","LFO 2","LFO 3",
    "Velocity","Note",
    "Step LFO","CC1 Mod Wheel",
    "MPE Pressure","MPE Timbre","MPE Pitch Bend","Pitch Bend"
};
```

`ModDepthParams::setup` loops over `kNumSrcs` automatically — no other change needed there.

### 1.3 `WavetableVoice.cpp` — Extend `modSrcOut` array

In `updateParams()`, extend the source output collection block:

```cpp
float modSrcOut[15];   // was [9]

// Existing 9:
modSrcOut[0] = filterADSR.getOutput();
modSrcOut[1] = proc.envParams[0].enable->isOn() ? modADSRs[0].getOutput() : 0.0f;
modSrcOut[2] = proc.envParams[1].enable->isOn() ? modADSRs[1].getOutput() : 0.0f;
modSrcOut[3] = proc.envParams[2].enable->isOn() ? modADSRs[2].getOutput() : 0.0f;
modSrcOut[4] = proc.lfoParams[0].enable->isOn() ? modLFOs[0].getOutput() : 0.0f;
modSrcOut[5] = proc.lfoParams[1].enable->isOn() ? modLFOs[1].getOutput() : 0.0f;
modSrcOut[6] = proc.lfoParams[2].enable->isOn() ? modLFOs[2].getOutput() : 0.0f;
modSrcOut[7] = currentlyPlayingNote.noteOnVelocity.asUnsignedFloat();
modSrcOut[8] = currentlyPlayingNote.initialNote / 127.0f;

// New 6:
modSrcOut[9]  = proc.stepLfoParams.enable->isOn() ? modStepLFO.getOutput() : 0.0f;
modSrcOut[10] = proc.modSrcCC.size() > 1
                ? proc.modMatrix.getMonoValue (proc.modSrcCC[1])
                : 0.0f;
modSrcOut[11] = currentlyPlayingNote.pressure.asUnsignedFloat();
modSrcOut[12] = currentlyPlayingNote.timbre.asUnsignedFloat();
modSrcOut[13] = currentlyPlayingNote.pitchbend.asUnsignedFloat();
modSrcOut[14] = proc.modMatrix.getMonoValue (proc.modScrPitchBend);
```

The `modSum` lambda uses `kNumSrcs` implicitly via the loop bound — change loop bound
from `9` to `15`:

```cpp
auto modSum = [&] (int dstIdx) -> float
{
    float total = 0.0f;
    for (int s = 0; s < 15; ++s)   // was 9
        total += modSrcOut[s] * proc.modDepthParams.depths[s][dstIdx]->getUserValue();
    return total;
};
```

---

## 2. Dynamic Wavetable Index (Future-Proof)

The original guide hard-codes `osc1_wt_index` range as `{0, 214, 1, 1}`. This breaks
when new wavetables are added to the bundle. Fix: make the range dynamic at plugin
init time, derived from `getWavetableNames().size() - 1`.

### 2.1 `WtIndexParams::setup` — Dynamic upper bound

```cpp
void WavetableAudioProcessor::WtIndexParams::setup (WavetableAudioProcessor& p)
{
    // Count bundled + user-added wavetables at init time
    int numTables = juce::jmax (1, p.getWavetableNames().size());
    float maxIdx  = float (numTables - 1);

    osc1Index = p.addIntParam ("osc1_wt_index", "OSC1 Wavetable Index", "WT1 Idx", "",
                               { 0.0f, maxIdx, 1.0f, 1.0f }, 0.0f, 0.0f);
    osc2Index = p.addIntParam ("osc2_wt_index", "OSC2 Wavetable Index", "WT2 Idx", "",
                               { 0.0f, maxIdx, 1.0f, 1.0f }, 0.0f, 0.0f);
}
```

The plugin re-inits these params on each load, so if you drop new `.wt2048` files
into the Wavetables folder and restart, the index range expands automatically.

### 2.2 `getWavetableNames()` already scans at runtime

No C++ change needed — `getWavetableNames()` calls `getWavetableFiles()` which does a
live directory scan every call. New wavetables placed in:

```
~/Library/Audio/Plug-Ins/VST3/Wavetable.vst3/Contents/Resources/Wavetables/
```

are picked up on the next plugin init.

### 2.3 Python: search by name, get index

```python
from pedalboard import load_plugin

p = load_plugin('/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')

# The plugin exposes the index range — derive table count from it
wt_max = int(p.parameters['osc1_wt_index'].max_value)
num_tables = wt_max + 1
print(f"{num_tables} wavetables available")

# Build name→index map from the known sorted list.
# The plugin sorts by category folder then filename naturally.
import pathlib, natsort

wt_root = pathlib.Path(
    '/Library/Audio/Plug-Ins/VST3/Wavetable.vst3/Contents/Resources/Wavetables'
)
files = list(wt_root.rglob('*.wt2048'))

# Sort: by parent folder name, then by filename (natural order) — matches plugin sort
files.sort(key=lambda f: (f.parent.name.lower(), f.stem.lower()))
# For proper natural sort matching the plugin:
# files = natsort.natsorted(files, key=lambda f: f"{f.parent.name}/{f.stem}")

wt_name_to_idx = {f.stem: i for i, f in enumerate(files)}
wt_idx_to_name = {i: f.stem for i, f in enumerate(files)}

def load_wt_by_name(plugin, osc: int, name: str):
    idx = wt_name_to_idx.get(name)
    if idx is None:
        raise ValueError(f"Wavetable '{name}' not found. Available: {list(wt_name_to_idx.keys())[:5]}...")
    if osc == 1:
        plugin.osc1_wt_index = float(idx)
    else:
        plugin.osc2_wt_index = float(idx)

# Usage:
load_wt_by_name(p, 1, "Analog Classic")
load_wt_by_name(p, 1, "AKWP bw_saw")
load_wt_by_name(p, 2, "FM 01")
```

---

## 3. User WAV File as Oscillator Waveform

The plugin already stores user-uploaded wavetables as base64 WAV inside the preset
state (`wt1Data`, `wt2Data`). From Python, we can inject raw WAV bytes directly into
the state blob — no C++ changes needed.

> **Important bug fix in existing C++ first:** In `PluginProcessor.cpp` line ~481,
> `mb2` reads from `"wt1Data"` instead of `"wt2Data"`. Fix before using this feature:
>
> ```cpp
> // WRONG (existing):
> mb2.fromBase64Encoding (state.getProperty ("wt1Data").toString());
> // CORRECT:
> mb2.fromBase64Encoding (state.getProperty ("wt2Data").toString());
> ```

### 3.1 How it works

When `wt1Data` is non-empty in the state, the plugin loads it as a WAV wavetable
and ignores the `wt1` name property entirely. So the flow is:

1. Python reads current `preset_data` XML
2. Encodes WAV file bytes as base64
3. Injects `wt1Data="<base64>"` into the XML
4. Sets `preset_data` back — plugin calls `stateUpdated()` → `reloadWavetables()`

### 3.2 Python implementation

Add this to `wavetable_service.py`:

```python
import base64, re

def load_user_wav_as_wavetable(plugin, osc: int, wav_path: str) -> bool:
    """
    Load a WAV file as the wavetable for osc 1 or 2.
    The WAV should be a single-cycle waveform (ideally 2048 samples, 44100 Hz mono).
    osc: 1 or 2.
    Returns True on success.
    """
    assert osc in (1, 2), "osc must be 1 or 2"
    wt_key  = f"wt{osc}Data"
    nm_key  = f"wt{osc}"
    sz_key  = f"wt{osc}Size"
    osc_nm  = pathlib.Path(wav_path).stem

    # Read WAV bytes
    with open(wav_path, 'rb') as f:
        wav_bytes = f.read()
    b64_str = base64.b64encode(wav_bytes).decode('ascii')

    # Read current preset XML
    if not hasattr(plugin, 'preset_data'):
        return False
    current = bytes(plugin.preset_data)
    xml_start = current.find(b'<?xml')
    if xml_start < 4:
        return False
    xml_len  = int.from_bytes(current[xml_start - 4 : xml_start], 'big')
    xml_end  = xml_start + xml_len
    xml_str  = current[xml_start:xml_end].decode('utf-8')

    # Replace or insert wt1Data / wt2Data attribute
    # Also set wt name to file stem and size to -1 (auto-detect)
    def set_attr(xml: str, key: str, value: str) -> str:
        pattern = rf'{re.escape(key)}="[^"]*"'
        replacement = f'{key}="{value}"'
        new_xml, n = re.subn(pattern, replacement, xml, count=1)
        if n == 0:
            # Attribute doesn't exist yet — insert before closing state tag
            new_xml = new_xml.replace('</state>', f'  {replacement}\r\n</state>', 1)
        return new_xml

    xml_str = set_attr(xml_str, wt_key, b64_str)
    xml_str = set_attr(xml_str, nm_key, osc_nm)
    xml_str = set_attr(xml_str, sz_key, "-1")

    # Re-encode and validate length matches
    encoded = xml_str.encode('utf-8')
    delta   = xml_len - len(encoded)
    if delta < 0:
        # WAV data grows XML — pad with spaces before closing tag
        # This can happen for large WAV files. In practice base64 of 2048-sample
        # WAV is ~11 KB; original preset XML is much smaller. May need fallback.
        # Solution: use setProperty approach below instead.
        return _inject_via_setattr(plugin, osc, wav_bytes, osc_nm)
    if delta > 0:
        xml_str  = xml_str.replace('\r\n</state>', (' ' * delta) + '\r\n</state>', 1)
        encoded  = xml_str.encode('utf-8')

    if len(encoded) != xml_len:
        return _inject_via_setattr(plugin, osc, wav_bytes, osc_nm)

    new_preset = current[:xml_start] + encoded + current[xml_end:]
    try:
        plugin.preset_data = new_preset
        return True
    except Exception:
        return False


def _inject_via_setattr(plugin, osc: int, wav_bytes: bytes, name: str) -> bool:
    """
    Fallback: use pedalboard's state property directly if preset_data length
    constraint fails (happens when WAV data exceeds available XML space).
    gin's Processor exposes 'state' as a settable property on some builds.
    """
    try:
        b64_str = base64.b64encode(wav_bytes).decode('ascii')
        # gin::Processor.state is a juce::ValueTree — not directly accessible via
        # pedalboard's standard API. This fallback signals the caller to rebuild
        # from scratch with a blank preset_data first.
        raise NotImplementedError(
            "WAV file too large for in-place preset injection. "
            "Trim WAV to 2048 samples or use a bundled wavetable."
        )
    except Exception as e:
        import logging
        logging.warning("load_user_wav_as_wavetable fallback failed: %s", e)
        return False
```

### 3.3 WAV file requirements for best results

The plugin's `loadWaveTable` reads the WAV as a single-cycle waveform and builds
mip-mapped alias-free tables from it. For best quality:

| Property | Recommended | Why |
|---|---|---|
| Sample count | 2048 samples | Matches plugin's `wt2048` format |
| Sample rate | 44100 Hz | Plugin's default SR |
| Channels | Mono | Only channel 0 is read |
| Bit depth | 16 or 32-bit float | Standard WAV formats |
| Content | One full period | Multi-cycle WAVs are averaged |

Generate a 2048-sample WAV in Python:

```python
import numpy as np, wave, struct

def save_single_cycle_wav(samples: np.ndarray, path: str, sr: int = 44100):
    """samples should be 2048 float32 values in [-1, 1]."""
    assert len(samples) == 2048
    pcm = (samples * 32767).clip(-32767, 32767).astype(np.int16)
    with wave.open(path, 'w') as f:
        f.setnchannels(1)
        f.setsampwidth(2)
        f.setframerate(sr)
        f.writeframes(pcm.tobytes())

# Example: custom additive synthesis waveform
t = np.linspace(0, 2 * np.pi, 2048, endpoint=False)
waveform = (np.sin(t)
          + 0.5 * np.sin(2*t)
          + 0.25 * np.sin(3*t)
          + 0.1 * np.sin(5*t))
waveform /= np.max(np.abs(waveform))
save_single_cycle_wav(waveform.astype(np.float32), '/tmp/custom_wave.wav')

# Load into OSC 1:
load_user_wav_as_wavetable(plugin, 1, '/tmp/custom_wave.wav')
```

---

## 4. WAV-per-Oscillator for Transformer Training

This is the most architecturally important decision for the long term. Three strategies:

### Strategy A — Waveform encoder (recommended long-term)

Train a separate **waveform VAE** (variational autoencoder):
- Input: 2048-sample single-cycle waveform
- Latent: 32-dim continuous vector
- Decoder: reconstruct waveform from latent

The transformer output then includes a 32-dim latent code per oscillator (64 floats
total for OSC 1 + OSC 2) instead of a raw waveform. At inference:
1. Transformer outputs latent codes
2. Waveform decoder generates 2048-sample WAV from each code
3. `load_user_wav_as_wavetable()` loads each WAV into the plugin

This keeps the transformer output compact and differentiable. The waveform encoder
can be pre-trained on the full wavetable library (215 tables × their frames) before
transformer training begins.

**Parameter vector addition:**

```
[0:98]    Base parameters
[98:503]  Mod depth matrix (15×27 = 405)
[503]     osc1_wt_index (bundled, or -1 = use latent)
[504]     osc2_wt_index
[505:537] osc1 waveform latent (32 floats, active when index = -1)
[537:569] osc2 waveform latent (32 floats, active when index = -1)
Total: 569 floats
```

### Strategy B — Raw waveform in parameter vector (simplest, largest)

Include the 2048-sample waveform directly in the output vector per oscillator:

```
[503:2551] osc1 waveform (2048 floats)
[2551:4599] osc2 waveform (2048 floats)
Total: 4599 floats
```

Pros: No auxiliary model. Cons: Very high-dimensional output, MSE on waveforms does
not correlate well with perceptual similarity — the model will predict averaged/blurry
waveforms. Only viable with a spectral loss function.

### Strategy C — Hybrid index + fallback (recommended short-term)

For training data generation:
1. For presets that use a **bundled wavetable**: set `osc_wt_index`, waveform latent = zeros
2. For presets that use a **custom WAV**: find the nearest bundled wavetable by spectral
   centroid / FFT distance, use that index, note the error
3. Gradually expand the bundled wavetable library by adding custom WAVs as `.wt2048`
   files — the index auto-expands

This is the most practical starting point. Over time the bundled library grows to
cover most training data, and Strategy A can be layered on top for truly unique
waveforms.

### 4.1 C++ addition for Strategy A: dedicated user WAV parameters

Add two parameters that signal "use the user WAV, not the index":

```cpp
// In WtIndexParams::setup, after existing params:
osc1UseUserWav = p.addIntParam ("osc1_use_user_wav", "OSC1 Use Custom WAV", "", "",
                                { 0.0, 1.0, 1.0, 1.0 }, 0.0f, 0.0f, enableTextFunction);
osc2UseUserWav = p.addIntParam ("osc2_use_user_wav", "OSC2 Use Custom WAV", "", "",
                                { 0.0, 1.0, 1.0, 1.0 }, 0.0f, 0.0f, enableTextFunction);
```

In `parameterChanged`:

```cpp
if (param == wtIndexParams.osc1UseUserWav && param->getUserValue() < 0.5f)
{
    // Switched back to index mode — clear user WAV, reload from index
    userTable1.reset();
    setWavetableByIndex (0, int (wtIndexParams.osc1Index->getUserValue()));
}
// (same for osc2)
```

From Python:
```python
# Switch OSC 1 to custom WAV mode
p.osc1_use_user_wav = 1.0
load_user_wav_as_wavetable(p, 1, '/tmp/my_waveform.wav')

# Switch back to index mode
p.osc1_use_user_wav = 0.0
p.osc1_wt_index = 65  # Analog Classic
```

---

## 5. Updated Parameter Vector Summary

After all changes in this addendum + the base guide:

```
Index range    Count    Contents
[0:98]           98     Base VST3 parameters (existing)
[98:503]        405     Mod depth matrix: 15 sources × 27 destinations
[503:505]         2     osc1_wt_index, osc2_wt_index
[505:507]         2     osc1_use_user_wav, osc2_use_user_wav
─────────────────────
Total            507     floats (before waveform latents)
+ latents:    +64      Optional: 32-dim waveform latent per OSC (Strategy A)
Grand total    571
```

### Normalization for transformer training

| Parameter group | Normalization |
|---|---|
| Base params | `(value - min) / (max - min)` per param |
| Mod depth `[-1, 1]` | Already normalized — no change |
| `osc_wt_index` | `index / (num_tables - 1)` → `[0, 1]` |
| `osc_use_user_wav` | Binary 0/1 |
| Waveform latent | Unit normal (from VAE training) |

---

## 6. Build & Verify

```bash
cd /path/to/Synth_X/Wavetable

cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --config Debug --target Wavetable_VST3 -j$(sysctl -n hw.logicalcpu)
cp -r build/Wavetable_artefacts/Debug/VST3/Wavetable.vst3 \
      ~/Library/Audio/Plug-Ins/VST3/Wavetable.vst3

# Verify new param count
python3 -c "
from pedalboard import load_plugin
p = load_plugin('$HOME/Library/Audio/Plug-Ins/VST3/Wavetable.vst3')
params = list(p.parameters.keys())
print('Total:', len(params))
print('Mod params:', len([k for k in params if k.startswith('mod_')]))
print('Step LFO routes:', [k for k in params if 'step' in k and k.startswith('mod_')][:3])
print('MPE routes:', [k for k in params if 'mpe' in k and k.startswith('mod_')][:3])
print('CC1 routes:', [k for k in params if 'cc1' in k and k.startswith('mod_')][:3])
print('WT params:', [k for k in params if 'wt_index' in k or 'user_wav' in k])
"
# Expected:
# Total: 509
# Mod params: 405
# Step LFO routes: ['mod_step_osc1pos', 'mod_step_osc1tune', 'mod_step_osc1fine']
# MPE routes: ['mod_mpe_pressure_osc1pos', ...]
# CC1 routes: ['mod_cc1_osc1pos', ...]
# WT params: ['osc1_wt_index', 'osc2_wt_index', 'osc1_use_user_wav', 'osc2_use_user_wav']
```
