# Beamforming and DOA Estimation for Acoustic Arrays

Demonstration of beamforming and direction-of-arrival (DOA)
estimation for array processing, with an emphasis on microphone arrays,
acoustic sensing, and spatial audio capture. The notation and examples follow the
[PySDR Beamforming & DOA chapter](https://pysdr.org/content/doa), adapted here
for acoustic arrays.

## Notebooks

- `Delay_and_Sum.ipynb`: conventional beamforming and a baseline DOA scan.
- `MVDR.ipynb`: adaptive Capon/MVDR spatial spectrum.
- `LCMV.ipynb`: constrained adaptive beamforming with look and null directions.
- `MUSIC.ipynb`: high-resolution subspace DOA estimation.

## Acoustic Array Model

For a uniform linear microphone array with $N_r$ sensors, the narrowband far-field
snapshot model is

$$ \mathbf{y}[n] = \mathbf{s} \cdot x[n] + \mathbf{v}[n] $$

where $X$ is the $N_r \times N$ received snapshot matrix, $x$ is the source
signal row vector, $s$ is the $N_r \times 1$ steering vector, and $n$ is noise.

For adjacent sensor spacing $d$ measured in wavelengths, the phase shift at the
$k$th sensor is:

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

The spatial covariance matrix is

$$
R = \frac{1}{N}X X^H
$$

where $N$ is the number of time samples. In Python:

```python
R = (X @ X.conj().T) / X.shape[1]
```

The notebooks use this narrowband far-field model throughout.

## Algorithm Math

### Delay-and-Sum

Delay-and-sum phase-aligns a look direction and sums the sensors. In PySDR
notation, the conventional weights are the steering vector for the look angle:

$$
w = s(\theta)
$$

$$
y = w^H X
$$

$$ P_{DAS}(\theta) = \text{var}(y) $$

Delay-and-sum is simple, robust, and intuitive, but its angular resolution is
limited by array aperture and sidelobe behavior.

### MVDR / Capon

MVDR minimizes output power while preserving unit response in the scan direction:

$$
\min_w \; w^H R w
$$

$$
\text{subject to} \quad w^H s = 1
$$

The weights and spectrum are

$$
w_{mvdr} = \frac{R^{-1}s}{s^H R^{-1}s}
$$

$$
P_{MVDR}(\theta) = \frac{1}{s^H R^{-1}s}
$$

MVDR can suppress interference better than delay-and-sum, but it depends on a
good covariance estimate and is sensitive to steering mismatch.

### LCMV

LCMV extends MVDR to multiple linear constraints:

$$
\min_w \; w^H R w
$$

$$
\text{subject to} \quad C^H w = f
$$

with solution

$$
w_{lcmv} = R^{-1} C (C^H R^{-1} C)^{-1} f
$$

$C$ contains steering vectors for constrained directions, and $f$ is the desired
response. For example, $f = [1, 0, 0]^T$ can preserve a talker direction while
forcing nulls toward two known interferers. PySDR also shows a multi-look case
such as $f = [1, 1, 0, 0]^T$.

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
  range.
- More microphones increase aperture and usually improve angular resolution.
- More snapshots improve covariance estimates for MVDR, LCMV, and MUSIC.
- A straight-line array has front/back ambiguity unless the geometry or signal
  model adds more information.
