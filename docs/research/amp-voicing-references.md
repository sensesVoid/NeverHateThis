# Amp voicing without captures — references

Purpose: we have no real amplifiers or cabinets to record, so the amp tier is built from circuit-informed DSP and tuned by ear against reference recordings. This is the reading list behind that choice, and what we take from each source.

## Resulting approach

Per voicing: input HPF → preamp (asymmetric waveshaper or triode stage) → passive tone stack → power amp (sag, presence, output-transformer low-pass) → parametric cabinet. No captures, no neural models, no NAM import.

The amp is a composition of four known blocks. The literature below covers each one, and the cabinet can be modelled without an IR by using the speaker's resonance and inductance.

## Amp and tone stack

| Source | What to take | Use |
|---|---|---|
| D. T. Yeh, J. S. Abel, J. O. Smith III, *Discretization of the '59 Fender Bassman Tone Stack*, DAFx-06 — <https://ccrma.stanford.edu/~dtyeh/tonestack/> | Third-order transfer function, bilinear transform, direct-form-II implementation. The same topology with different component values gives Fender, Marshall and Vox tone stacks. | Core of the tone stack. Re-implement from the equations. |
| J. Ragland, *Digital Simulation and Recreation of a Vacuum Tube Guitar Amp*, Auburn University, 2020 — <https://etd.auburn.edu/bitstream/handle/10415/7112/Digital%20Simulation%20and%20Recreation%20of%20a%20Vacuum%20Tube%20Guitar%20Amp.pdf> | Full amp recreation: tube stages, lookup-table vs. iterative solver, tone-stack discretization, listening survey. | End-to-end sanity check and test plan. |
| M. Macak, J. Schimmel, *Real-Time Guitar Tube Amplifier Simulation using an Approximation of Differential Equations*, DAFx-10 — <http://dafx10.iem.at/papers/MacakSchimmel_DAFx10_P12.pdf> | Koren triode model, differential-equation approximation, real-time cost. | Preamp stage when a waveshaper is not enough. |
| N. Koren, *Improved Vacuum Tube Models* — <http://www.normankoren.com/Audio/Tubemodspice_article.html> | Triode/pentode equations used by most white-box sims. | Tube curve reference. |
| W. M. Leach, *SPICE Models for Vacuum-Tube Amplifiers*, JAES 1995 | Alternative tube model; the 12AX7 values are widely reproduced. | Cross-check against Koren. |
| *DDSP guitar amp*, arXiv:2408.11405 — <https://arxiv.org/pdf/2408.11405> | Modular decomposition into preamp, tone stack, power amp and output transformer; DSP-inspired blocks with far fewer operations than black-box models. | Architecture reference, not a dependency. |
| T. Vanhatalo et al., *A Review of Neural Network-Based Emulation of Guitar Amplifiers*, 2022 — <https://www.mdpi.com/2076-3417/12/12/5894> | Survey of white-box, grey-box and black-box approaches; where waveshaping plus filters is sufficient and where it is not. | Choosing the simplest model that passes listening tests. |

## Cabinet without an IR

| Source | What to take | Use |
|---|---|---|
| D. T. Yeh, B. Bank, *Nonlinear Modeling of a Guitar Loudspeaker Cabinet*, DAFx-08 — <https://home.mit.bme.hu/~bank/publist/dafx08cab.pdf> | Linear parallel-filter model of the speaker, plus low-frequency nonlinear behavior; measurement method. | Parametric cab structure. |
| S. Grimm, *Hybrid Modeling of Guitar Cabinets Using FIR Filters and Neural Networks*, DAGA 2026 — <https://pub.dega-akustik.de/DAGA_2026/files/upload/paper/154.pdf> | 400–600 FIR taps are perceptually near-indistinguishable from the real cabinet; nonlinear cab modelling adds little for typical playing. | Justifies a linear cab model and bounds its length. |
| H. Keller, *A Tube Amp Modeling Project*, 2024 — <https://gitec-forum-eng.de/wp-content/uploads/A-Tube-Amp-Modeling-Project-V1.0.2.pdf> | Practical IIR cabinet: Thiele-Small resonance peak (fs, Qts), inductance high-shelf, low-pass; all parameters exposed as controls. | Default cab implementation. |
| Z² DSP, *Speaker Cab Modelling Part II* — <https://z2dsp.com/2018/07/10/impulse-response-modelling> | How to capture an IR if we ever get access to a cab. | Future content pipeline. |
| Sound on Sound, *Making Great Guitar Cab IRs* — <https://www.soundonsound.com/techniques/colin-cartmell-making-great-guitar-cab-irs> | Practical capture advice: flat amp, sweep, deconvolution, 20–41 ms trim. | Future capture guide. |
| Celestion, *Thinking of using Thiele/Small parameters to design a guitar speaker cab?* — <https://celestion.com/blog/thinking-of-using-thiele-small-parameters-to-design-a-guitar-speaker-cab-think/> | T/S parameters are a rough guide for guitar cabs; the box matters less than for hi-fi. | Sets expectations for the parametric model. |

## Tuning by ear

No captures means voicing is an iterative listening process, not a training run:

1. Render a fixed DI pack (clean single notes, chords, palm mutes, double stops) through the candidate voicing.
2. A/B against isolated guitar stems the owner supplies; keep them out of the repo.
3. Compare 1/3-octave spectra and adjust tone-stack values and gain staging, then re-render.
4. Keep a small golden set per voicing so changes do not regress.

The reference recordings are for tuning only. Nothing derived from a commercial recording is shipped.

## Deliberately avoided

- **NAM / WaveNet captures.** No data to train on, and the real-time cost is a multi-month project.
- **GPL code.** Guitarix, BYOD, CAPS and RP2040-DSP are useful to study but copying any of them makes the app GPL. We re-implement from papers.
- **Faust-generated code.** The toolchain's licensing and generated-code terms add complexity we do not need.
- **"Free" IR packs with unclear terms.** If an IR ever ships, it needs a per-file license and a signed manifest.

## License notes

Circuit equations and topologies are not copyrightable; a re-implementation from a paper is ours. Code from GPL projects is read-only reference, never copied. Every DSP file that follows a paper carries a comment naming the source.
