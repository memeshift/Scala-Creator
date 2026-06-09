# Pitch Detection: Method Comparison and Design Notes

This document covers the pitch detection methods evaluated for Scala Creator, explains why NSDF was chosen as the current primary engine, and lays out the known limitations and recommended development path for handling a broader range of instrument types. It is intended for developers, researchers, or anyone curious about the tradeoffs involved.

---

## TL;DR

- **The current approach (NSDF with global-maximum peak selection) is the right choice for most gamelan metallophones, but the wrong tool for the hardest cases.** Time-domain periodicity methods (autocorrelation, NSDF/MPM, YIN, pYIN) assume the sound is a repeating waveform. Struck metallophones, bonang, gongs and bells are inharmonic — they have no true period — so these methods are prone to wrong-partial errors on the most acoustically complex material.
- **For cents-precise tuning of inharmonic idiophones, a frequency-domain spectral peak-picker on the lowest strong partial — refined with parabolic interpolation — is the perceptually and physically correct method.** Research shows the perceived pitch of gamelan bars and gongs tracks the lowest strong partial, which makers tune precisely, while upper partials sit at non-integer ratios.
- **Recommended development path:** keep NSDF as a fast first-pass and confidence cross-check; add an interpolated-FFT spectral analyzer as the primary engine for inharmonic uploads; expose a small number of UI parameters (frequency range, partial-selection mode, confidence threshold, FFT/window size); treat CREPE/ML as an optional cross-check, not core.

---

## Five methods, two families

**Periodicity / time-domain methods** — autocorrelation, NSDF (McLeod Pitch Method / MPM), YIN, pYIN — ask: "At what time-lag does the waveform best repeat?" They work well when the sound is harmonic, with partials at integer multiples of f0. Struck idiophones are inharmonic: their overtones are not integer multiples, so the waveform never truly repeats. The algorithm has nothing stable to lock onto.

**Frequency-domain methods** — interpolated FFT (QIFFT), cepstral analysis — ask: "Where is the energy in the spectrum?" They make no periodicity assumption. For an inharmonic idiophone you can directly read the frequency of the lowest strong partial and refine it to sub-cent precision with interpolation.

**Data-driven methods** — CREPE and variants — use a neural network trained on large corpora of pitched audio. State-of-the-art accuracy on harmonic and vocal material; untested and likely unreliable on inharmonic idiophones, for which they were not trained.

---

## Method-by-method

### 1. NSDF / McLeod Pitch Method — current implementation

Computes a Normalized Square Difference Function (a normalized relative of autocorrelation that stays in the range −1…+1 regardless of amplitude), then picks a peak. The normalization reduces edge artifacts and provides a "clarity" measure indicating how tone-like the signal is. Runs cheaply in the browser.

- **Accuracy:** ~1 cent on clear harmonic/monophonic tones (McLeod & Wyvill, 2005).
- **Inharmonic robustness:** Weak. Improves over raw autocorrelation but still a periodicity method. The current implementation uses global-maximum peak selection, which is more octave-error-prone than canonical MPM (which picks the *first* major key maximum). This choice was deliberate: for rich metallic gamelan sounds, the fundamental's NSDF peak is reliably the strongest, and the global maximum avoids spurious low-frequency candidates. For other material this assumption may not hold.
- **Soft/muted sounds:** The clarity measure helps reject noise, but fast-decaying sounds (bamboo instruments, heavily damped materials) lose periodicity quickly as upper partials die at different rates.
- **Browser cost:** Cheap — O((W+w)·log(W+w)) via FFT-based autocorrelation. Runs easily in Web Audio.

### 2. YIN and pYIN

YIN computes a difference function, then a cumulative-mean-normalized version, and picks the first dip below a threshold (~0.15). pYIN runs YIN across a distribution of thresholds and Viterbi-decodes an HMM for a smooth pitch track.

- **Accuracy:** ±1–2 cents on clear sustained tones.
- **Inharmonic robustness:** Still a periodicity method — fails "when there are too many high-energy upper partials," which is the inharmonic idiophone case. pYIN's HMM benefits are limited for single struck notes.
- **When to consider:** pYIN may improve tracking on sustained non-percussive instruments from other traditions.

### 3. Interpolated FFT (QIFFT) — recommended primary engine for inharmonic material

Take the FFT, find the magnitude peak bin, fit a parabola through the peak and its neighbors on a dB magnitude scale to estimate the true peak between bins. Zero-padding improves accuracy further.

- **Accuracy:** Better than 0.1% of frequency with adequate zero-padding. Well below 1 cent for a clean partial.
- **Inharmonic robustness:** This is the key advantage. No periodicity assumption. The lowest strong partial is read directly — which is what carries perceived pitch for gamelan bars and gongs. The well-known weakness (picking an upper partial instead of the fundamental) is solvable with a frequency-range constraint and lowest-strong-peak logic.
- **Muted/soft robustness:** Good — a clear spectral peak survives even in quiet recordings; use a long analysis window.
- **Browser implementation note:** The Web Audio `AnalyserNode` default FFT is too coarse for cents-level tuning. A custom zero-padded FFT is needed.

### 4. Cepstral pitch detection

Takes the FFT, log magnitude, then an inverse FFT. The cepstrum reveals the periodic spacing of harmonics; the peak quefrency corresponds to the pitch period. Primarily used in speech processing.

- **Inharmonic robustness:** Poor by design. Looks for evenly spaced (harmonic) peaks. Inharmonic idiophones have unevenly spaced partials with no clean cepstral peak. Not recommended for this use case.

### 5. CREPE (deep neural network)

A six-layer convolutional network operating on 1024-sample, 16 kHz frames, outputting probabilities over 360 pitch bins (each 20 cents wide). Released 2018 by Kim, Salamon, Li & Bello.

- **Accuracy on harmonic material:** State-of-the-art — 0.995 raw pitch accuracy within 10 cents on RWC-Synth.
- **Inharmonic robustness:** Unverified. Trained on vocal and harmonic instrumental audio; authors note "noisy results for uncommon instruments." Its 20-cent bins require interpolation; it may inherit wrong-partial ambiguity with no guarantee of cents-level correctness on bonang, gongs, or bells.
- **Browser cost:** Heavy. Full model ~22M parameters / ~89 MB. Tiny CREPE (via TF.js / ml5.js) runs in the browser but makes more octave errors. Not suitable as the primary engine.
- **Recommendation:** Optional, clearly-labeled cross-check only.

---

## The decisive evidence on inharmonic instruments

**Gamelan bars (saron, gender, slenthem)** vibrate like free-free bars with modal ratios near 1.00 : 2.76 : 5.40 : 8.93 — strongly non-integer. The perceived pitch is the lowest transverse mode.

**McLachlan, Marco & Wilson (2013, *Frontiers in Psychology*, 4:768):** the lowest-frequency partials of the gong, metallophone and xylophone notes studied were tuned within 2 Hz of each other, while higher-order inharmonic partials varied widely. The paper states that "gamelan instruments are tuned by the frequency of their first partial" and that the auditory system "rapidly primes only the lowest frequency partial of inharmonic timbres." This is the strongest single justification for spectral peak-picking on the lowest strong partial.

**Bossed gongs / bonang:** the central boss forces a near-2:1 relationship between the two principal axisymmetric modes. For some bossed gongs the pitch-bearing partial may be the 2nd or 3rd partial — a pure "lowest peak" rule needs a frequency-range constraint.

**Bells (the exception):** the perceived strike note is a virtual pitch built from the nominal, superquint and octave-nominal partials, approximately equal to nominal ÷ 2 for a mid-size bell. A naive "lowest strong peak" reads the hum partial and is an octave or two low. A dedicated bell/strike-note mode is needed.

**Bamboo instruments (jegog, angklung, bamboo saron):** faster decay and lower spectral richness than metal. NSDF loses signal quickly in the decay; a long-window FFT on the earliest sustained portion is likely more reliable. These instruments may also be closer to harmonic than metallophones, reducing the inharmonicity problem.

**Non-percussive pitched instruments:** sustained tones from other traditions (plucked, bowed, blown) tend to be more harmonic, which works in NSDF's favor — but the frequency range and window size may need adjustment. The detection range controls already address much of this.

---

## Parameters that belong in the UI

The Detection Range (min/max Hz) control already exists and is the single most valuable tool for reducing wrong-partial errors. Additional parameters worth exposing:

- **FFT/window size and zero-padding factor** — trades frequency resolution against time localization. Relevant for a future spectral engine.
- **Confidence/clarity threshold** — NSDF clarity, YIN threshold, or CREPE confidence; rejects ambiguous detections.
- **Partial / strike-note mode** — "lowest strong partial" (gamelan bars/gongs) vs. "bell strike note (nominal ÷ 2)." Handles the bell exception explicitly.
- **Analysis time window** — lets the user select the post-attack steady portion; the attack transient is the noisiest part for all methods.

---

## Recommended development path

1. **Add an interpolated-FFT spectral analyzer as the primary engine for uploaded samples.** Use a large zero-padded FFT on a user-selectable post-attack window, find spectral peaks, select the lowest strong partial within the settable frequency range, refine with parabolic interpolation on dB magnitude. Keep NSDF as a fast first-pass and confidence cross-check.

2. **Add the UI parameters that prevent the dominant error class.** Expose min/max frequency range (already done), FFT/window size, confidence threshold, and instrument-type / partial-selection mode including a bell strike-note option. Provide per-instrument presets.

3. **Add a confirmation layer.** Show the detected spectrum with the chosen partial highlighted, plus the full partial series and ratios. For ambiguous cases, run a second method and flag disagreement.

4. **Treat CREPE/ML as optional and clearly labeled.** If added, via TF.js/ml5.js, present it as an experimental cross-check — not the source of truth for tuning files where cents precision matters.

---

## Accuracy caveats

All published accuracy figures (±1 cent for MPM, ±1–2 cents for YIN, CREPE 10-cent-threshold numbers) come from harmonic or vocal test sets. They do not transfer directly to inharmonic idiophones, where the harder problem is *which partial to report*, not how precisely to measure it.

The perceived pitch of inharmonic instruments is also genuinely ambiguous. Gamelan instruments are often deliberately tuned with slight detuning between paired instruments, and tuning may deviate substantially from any theoretical scale. The "true pitch" encoded in a `.scl` file is a modeling choice: the lowest strong partial is the best-supported one for most metallophones, but bossed gongs and bells are exceptions.

---

## References

- McLeod, P. & Wyvill, G. (2005). A smarter way to find pitch. *Proceedings of the International Computer Music Conference.*
- de Cheveigné, A. & Kawahara, H. (2002). YIN, a fundamental frequency estimator for speech and music. *JASA.*
- Mauch, M. & Dixon, S. (2014). pYIN: A fundamental frequency estimator using probabilistic threshold distributions. *ICASSP.*
- Kim, J.W., Salamon, J., Li, P. & Bello, J.P. (2018). CREPE: A convolutional representation for pitch estimation. *ICASSP.*
- McLachlan, N., Marco, D. & Wilson, S. (2013). Pitch salience of two simultaneous tones. *Frontiers in Psychology*, 4:768.
- Rossing, T.D. & Shepherd, R.B. (1982). Acoustics of gamelan instruments. *Percussive Notes.*
- Woodhouse, J. *Euphonics.* §3.2 (beam vibration modes).
- Camacho, A. & Harris, J.G. (2008). A sawtooth waveform inspired pitch estimator for speech and music. *JASA.* (SWIPE)
