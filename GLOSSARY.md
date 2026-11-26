# Lightcurve Pipeline

A Stimela recipe that takes calibrated MeerKAT target visibilities through flagging,
self-calibration and deep imaging, and then extracts lightcurves through transient imaging.

## Language

**Band**:
The receiver and centre frequency of an observation: UHF, L, or one of S0 to S4.
_Avoid_: sub-band (for S0 to S4)

**S-band**:
MeerKAT's 1.75–3.5 GHz receiver, which observes 875 MHz at a time at one of five centre
frequencies. Each of these is a band, S0 to S4.
_Avoid_: S-band (for one of S0 to S4)
