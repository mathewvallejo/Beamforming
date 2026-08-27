# Beamforming and DOA Estimation for Acoustic Arrays

This repo is a notebook-first introduction to beamforming and direction-of-arrival
(DOA) estimation for array processing, with an emphasis on microphone arrays,
acoustic sensing, and audio spatial capture. The notation follows the
[PySDR Beamforming & DOA chapter](https://pysdr.org/content/doa), adapted here
for acoustic arrays.

Each notebook is standalone: a student can download one `.ipynb`, run it in
Jupyter or Colab, and regenerate the plots without relying on shared helper
modules.

## Notebooks

- `Delay_and_Sum.ipynb`: conventional beamforming and a baseline DOA scan.
- `MVDR.ipynb`: adaptive Capon/MVDR spatial spectrum.
- `LCMV.ipynb`: constrained adaptive beamforming with look and null directions.
- `MUSIC.ipynb`: high-resolution subspace DOA estimation.

Install the small dependency set:

```bash
python3 -m pip install -r requirements.txt
```

## Acoustic Array Model

For a uniform linear microphone array with `Nr` sensors, the narrowband far-field
snapshot model is

```text
X = S(theta) x + n
```

where `X` is the `Nr x N` received snapshot matrix, `x` is the source signal row
vector, `n` is noise, and `S(theta)` is built from steering vectors. For one
source, this is simply `X = s x + n`, where `s` is `Nr x 1`.

For spacing `d` measured in wavelengths,

```text
s(theta) = [1,
            exp(2j pi d sin(theta)),
            exp(2j pi d 2 sin(theta)),
            ...,
            exp(2j pi d (Nr - 1) sin(theta))]^T
```

The sample covariance matrix is

```text
R = (1 / N) X X^H
```

where `N` is the number of time samples. In Python:

```python
R = (X @ X.conj().T) / X.shape[1]
```

In acoustic systems, this model is most accurate over narrow frequency bands.
Broadband audio is usually handled by short-time Fourier transforms or
filter-bank processing, applying narrowband beamforming per frequency bin.

## Algorithm Math

### Delay-and-Sum

Delay-and-sum phase-aligns a look direction and sums the sensors. In PySDR
notation, the conventional weights are the steering vector for the look angle:

```text
w = s(theta)
y = w^H X
P_DAS(theta) = var(y)
```

Some implementations divide `w` by `Nr` for unity-gain normalization; that does
not change a normalized DOA scan. Delay-and-sum is simple, robust, and intuitive,
but its angular resolution is limited by array aperture and sidelobe behavior.

### MVDR / Capon

MVDR minimizes output power while preserving unit response in the scan direction:

```text
minimize    w^H R w
subject to  w^H s = 1
```

The weights and spectrum are

```text
w_mvdr = R^-1 s / (s^H R^-1 s)
P_MVDR(theta) = 1 / (s^H R^-1 s)
```

MVDR can suppress interference better than delay-and-sum, but it depends on a
good covariance estimate and is sensitive to steering mismatch. The notebooks use
`np.linalg.pinv(R)` in the same spirit as PySDR's examples.

### LCMV

LCMV extends MVDR to multiple linear constraints:

```text
minimize    w^H R w
subject to  C^H w = f
```

with solution

```text
w_lcmv = R^-1 C [C^H R^-1 C]^-1 f
```

`C` contains steering vectors for constrained directions, and `f` is the desired
response. For example, `f = [1, 0, 0]^T` can preserve a talker direction while
forcing nulls toward two known interferers. PySDR also shows a multi-look case
such as `f = [1, 1, 0, 0]^T`.

### MUSIC

MUSIC is a subspace DOA estimator. After eigendecomposing the covariance,

```text
R = V_s Lambda_s V_s^H + V_n Lambda_n V_n^H
```

the steering vectors for true sources are ideally orthogonal to the noise
subspace `V_n`. The MUSIC scan metric is

```text
theta_hat = argmax_theta 1 / (s^H V_n V_n^H s)
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
