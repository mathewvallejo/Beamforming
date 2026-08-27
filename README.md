# Beamforming and DOA Estimation for Acoustic Arrays

Demonstration of beamforming and direction-of-arrival
(DOA) estimation for array processing, with an emphasis on microphone arrays,
acoustic sensing, and spatial audio capture.

## Notebooks

- `Delay_and_Sum.ipynb`: conventional beamforming and a baseline DOA scan.
- `MVDR.ipynb`: adaptive Capon/MVDR spatial spectrum.
- `LCMV.ipynb`: constrained adaptive beamforming with look and null directions.
- `MUSIC.ipynb`: high-resolution subspace DOA estimation.

## Acoustic Array Model

For a uniform linear array (ULA) of microphones with `M` sensors, the narrowband far-field
snapshot model is

```text
x[n] = A(theta) s[n] + v[n]
```

where `x[n]` is the `M x 1` microphone snapshot, `s[n]` contains the source
signals, `v[n]` is noise, and `A(theta)` is built from steering vectors.

For spacing `d` measured in wavelengths,

```text
a(theta) = [1,
            exp(j 2 pi d sin(theta)),
            ...,
            exp(j 2 pi d (M - 1) sin(theta))]^T
```

The sample covariance matrix is

```text
R_xx = (1 / N) X X^H
```

where `X` is the `M x N` snapshot matrix. In acoustic systems, this model is most
accurate over narrow frequency bands. Broadband audio is usually handled by
short-time Fourier transforms or filter-bank processing, applying narrowband
beamforming per frequency bin.

## Algorithm Math

### Delay-and-Sum

Delay-and-sum phase-aligns a look direction and averages the sensors:

```text
w_DAS(theta) = a(theta) / M
y[n] = w_DAS(theta)^H x[n]
P_DAS(theta) = w_DAS(theta)^H R_xx w_DAS(theta)
```

It is simple, robust, and intuitive, but its angular resolution is limited by
array aperture and sidelobe behavior.

### MVDR / Capon

MVDR minimizes output power while preserving unit response in the scan direction:

```text
minimize    w^H R_xx w
subject to  w^H a(theta) = 1
```

The weights and spectrum are

```text
w_MVDR(theta) = R_xx^-1 a(theta) / (a(theta)^H R_xx^-1 a(theta))
P_MVDR(theta) = 1 / (a(theta)^H R_xx^-1 a(theta))
```

MVDR can suppress interference better than delay-and-sum, but it depends on a
good covariance estimate and is sensitive to steering mismatch. Diagonal loading
is commonly used for stability.

### LCMV

LCMV extends MVDR to multiple linear constraints:

```text
minimize    w^H R_xx w
subject to  C^H w = f
```

with solution

```text
w_LCMV = R_xx^-1 C (C^H R_xx^-1 C)^-1 f
```

`C` contains steering vectors for constrained directions. For example,
`f = [1, 0, 0]^T` can preserve a talker direction while forcing nulls toward two
known interferers.

### MUSIC

MUSIC is a subspace DOA estimator. After eigendecomposing the covariance,

```text
R_xx = E_s Lambda_s E_s^H + E_n Lambda_n E_n^H
```

the steering vectors for true sources are ideally orthogonal to the noise
subspace `E_n`. The MUSIC pseudospectrum is

```text
P_MUSIC(theta) = 1 / (a(theta)^H E_n E_n^H a(theta))
```

MUSIC can produce very sharp DOA peaks, but it needs the number of sources and a
reliable covariance estimate.

## Practical Notes

- Half-wavelength spacing helps avoid grating lobes over the broadside scan
  range.
- More microphones increase aperture and usually improve angular resolution.
- More snapshots improve covariance estimates for MVDR, LCMV, and MUSIC.
- A straight-line array has front/back ambiguity unless the geometry or signal
  model adds more information.
- Real audio arrays are broadband, reverberant, and affected by calibration
  errors, so these notebooks should be read as clean first-principles demos.

