# Beamforming and DOA Estimation for Acoustic Arrays

Python implementations of common beamforming and direction-of-arrival (DOA)
estimation. Examples are primarily based on theory and notation presented in
[PySDR Beamforming & DOA chapter](https://pysdr.org/content/doa).

## Notebooks

- `Delay_and_Sum.ipynb`: conventional beamforming and a baseline DOA scan.
- `MVDR.ipynb`: adaptive Capon/MVDR spatial spectrum.
- `LCMV.ipynb`: constrained adaptive beamforming with multiple response constraints.
- `MUSIC.ipynb`: high-resolution subspace DOA estimation.

## Acoustic Array Model

For a uniform linear microphone array with $N_r$ sensors, the single-source
narrowband far-field received signal model is:

$$
X = s x + n
$$

where:
- $X$ is the $N_r \times N$ received signal matrix
- $s$ is the $N_r \times 1$ steering vector
- $x$ is the $1 \times N$ source signal row vector for N samples
- $n$ is the $N_r \times N$ noise matrix

For adjacent sensor spacing $d$ measured in wavelengths, the phase shift at the
element indexed by $k$ is:

$$
e^{2j\pi d k \sin(\theta)}
$$

where $k = 0, 1, \ldots, N_r - 1$. This gives the steering vector $s(\theta)$, with
phase shifts relative to the first element:

$$
s(\theta) =
\begin{bmatrix}
1 \\
e^{2j\pi d\sin(\theta)} \\
e^{2j\pi d \cdot 2\sin(\theta)} \\
\vdots \\
e^{2j\pi d (N_r - 1)\sin(\theta)}
\end{bmatrix}
$$

The spatial covariance matrix estimated from $X$ is

$$
R = \frac{1}{N}X X^H
$$

where $N$ is the number of time samples. In Python:

```python
R = (X @ X.conj().T) / X.shape[1]
```

The MVDR and MUSIC notebooks use the same idea with multiple source directions by
directly adding them:

```python
X = s1 @ x1 + s2 @ x2 + 0.1 * s3 @ x3
```

## Algorithms

### Delay-and-Sum

Delay-and-sum (DAS), also called conventional beamforming, phase-aligns a look direction and sums the sensors. In PySDR notation, the conventional weights are the steering vector for the look angle:

$$
w = s(\theta)
$$

The beamformer output is:

$$
X_{weighted}(\theta) = w^H X
$$

For DOA estimation, the look direction is swept over a range of angles and the output power is evaluated at each angle:

$$
P_{DAS}(\theta) = var(X_{weighted}(\theta))
$$

The estimated DOA corresponds to the angle producing the maximum output power.

### MVDR / Capon

MVDR minimizes output power while preserving unit response in the scan
direction. It can be used when $X$ contains multiple arrivals, but one MVDR
weight vector protects one steering direction at a time. During DOA scanning,
each value of $\theta$ creates a new steering vector $s(\theta)$ and a new
power value. If more than one direction needs to be preserved at the same time,
that becomes an LCMV problem.

$$
\min_w \; w^H R w
$$

$$
w^H s = 1
$$

where:
- $w$ is the beamforming weight vector
- $H$ means conjugate transpose
- $R$ is the spatial covariance matrix estimated from $X$
- $s$ is the steering vector for the current scan angle
- $w^H R w$ is the output power being minimized
- $w^H s = 1$ forces unity gain in the scan direction

The PySDR MVDR/Capon weights are:

$$
w_{mvdr} = \frac{R^{-1}s}{s^H R^{-1}s}
$$

where $R^{-1}$ is usually implemented with a pseudo-inverse in the notebooks:

```python
Rinv = np.linalg.pinv(R)
```

For DOA scanning, the power is represented as:

$$
P_{MVDR}(\theta) = \frac{1}{s^H R^{-1}s}
$$

where $P_{MVDR}(\theta)$ is the MVDR spatial spectrum value for the current
scan angle.

MVDR can suppress interference better than delay-and-sum, but it depends on a
good covariance estimate and is sensitive to steering mismatch.

### LCMV

LCMV extends MVDR to multiple linear constraints:

$$
\min_w \; w^H R w
$$

$$
C^H w = f
$$

with solution

$$
w_{lcmv} = R^{-1} C (C^H R^{-1} C)^{-1} f
$$

$C$ contains steering vectors for constrained directions, and $f$ is the desired
response. For example, $f$ can preserve one or more talker directions and can
also place nulls toward known interferers. The LCMV notebook follows the PySDR
style by simulating four received signals, then using $f = [1, 1]^T$ to place
unit-response constraints at two SOI directions.

### MUSIC

MUSIC is a subspace DOA estimator. After eigendecomposing the covariance,

$$
R = V_s \Lambda_s V_s^H + V_n \Lambda_n V_n^H
$$

the steering vectors for true sources are ideally orthogonal to the noise
subspace $V_n$. The MUSIC scan metric is

$$
\hat{\theta} = \arg\max_\theta \frac{1}{s^H V_n V_n^H s}
$$

MUSIC can produce very sharp DOA peaks, but it needs the number of sources and a
reliable covariance estimate.

## Practical Notes

- Half-wavelength spacing helps avoid grating lobes over the broadside scan
  range
- More microphones increase aperture and usually improve angular resolution
- More snapshots improve covariance estimates for MVDR, LCMV, and MUSIC
- A straight-line array has front/back ambiguity unless the geometry or signal
  model adds more information
