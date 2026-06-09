# Scala Creator

A browser-based tool for generating [Scala tuning files](https://www.huygens-fokker.org/scala/) from uploaded audio samples. Built for musicians, artists, and researchers working with microtonal and alternative tuning systems.

## What it does

Upload a batch of tonal audio files (WAV, AIF, AAC, and other common formats) and Scala Creator analyses the frequency relationships between them. From there, you can fine-tune individual pitches to correct for fundamental or harmonic discrepancies, then export the result as a `.scl` file ready for use in any Scala-compatible software.

All audio processing happens locally in the browser — no files are uploaded to a server.

## Status

Working prototype. Pitch detection is functional but accuracy is an ongoing area of development. Feedback on edge cases, unusual timbres, or workflow gaps is welcome. Issues and pull requests are welcome.

## How pitch detection works

The current implementation uses NSDF (Normalized Square Difference Function), a variant of autocorrelation tuned for sustained metallic tones. It works reliably on gamelan metallophones and most pitched instruments with a clear fundamental; it can struggle with fast-decaying sounds, large gongs and bells, and instruments with dense inharmonic spectra.

See [PITCH_DETECTION_NOTES.md](PITCH_DETECTION_NOTES.md) for a full comparison of the methods evaluated, the reasoning behind the current approach, and a recommended development path for improving accuracy across a wider range of instruments.

## Inspiration

This project is inspired by the work of [Latent Sonorities](https://www.latentsonorities.org) and Dr. Khyam Allami, and draws influence from the [Leimma](https://isartum.net/leimma) tool he developed with Counterpoint.

## License

TBD
