# Parametric Curve Parameter Recovery

Recovery of unknown parameters **θ**, **M**, and **X** for a 2D parametric curve from a point cloud of `(x, y)` observations. The full workflow is implemented in [`R&D_Task_Solved.ipynb`](R&D_Task_Solved.ipynb).

---

## Dataset

**File:** `xy_data.csv`

| Property | Value |
|----------|-------|
| Samples | 1,500 |
| Columns | `x`, `y` |
| Missing values | None |
| Parameter `t` | Not provided per row |

The dataset contains 1,500 planar coordinates sampled from an unknown parametric curve. Each row is an `(x, y)` pair; the underlying parameter `t` is latent and lies in the interval **6 ≤ t ≤ 60**. The scatter forms a smooth, oscillatory curve in the plane—consistent with a rotated parametric model with exponential amplitude modulation.

---

## Problem Statement

Given only the point cloud `(xᵢ, yᵢ)` for `i = 1 … 1500`, recover the three unknown parameters of the generating curve:

**Parametric model**

```
x(t) = t·cos(θ) − e^(M|t|)·sin(0.3t)·sin(θ) + X
y(t) = 42 + t·sin(θ) + e^(M|t|)·sin(0.3t)·cos(θ)
```

**Unknowns and search bounds**

| Parameter | Meaning | Bounds |
|-----------|---------|--------|
| θ | Rotation angle | (0°, 50°) → (0, 0.8727) rad |
| M | Exponential growth/decay rate | (−0.05, 0.05) |
| X | Horizontal offset | (0, 100) |

**Challenge:** This is an inverse problem. `t` is not observed, so standard regression on `(t, x, y)` is not possible. The task reduces to **fitting a parametric curve to a point cloud** by searching over `(θ, M, X)` and measuring how well the resulting curve explains the data.

---

## Methodology

### 1. Forward model

For candidate parameters `(θ, M, X)`, the curve is evaluated on a dense grid of `t` values from 6 to 60 (8,000 points) to capture the `sin(0.3t)` oscillation without aliasing.

### 2. Loss function (point-cloud → curve distance)

Because `t` is unknown for each data point, fit quality is measured as the **mean nearest-neighbor distance** from every data point to the model curve:

1. Build a dense reference curve from `(θ, M, X)`.
2. Construct a KD-tree on the curve points.
3. For each of the 1,500 data points, query the nearest point on the curve.
4. Minimize the average distance across all data points.

This metric treats the data as a noisy sample of the curve and is appropriate when parameter `t` is latent.

### 3. Global optimization (Differential Evolution)

A bounded **differential evolution (DE)** search explores the 3D parameter space `(θ, M, X)` without requiring gradients. To keep runtime practical (~10 s per run):

- **Phase 1 — coarse global search:** DE on 800 subsampled data points and a 4,000-point reference curve.
- **Phase 2 — full-resolution polish:** Nelder–Mead local refinement on all 1,500 points and the full 8,000-point curve.

### 4. Local refinement (Nelder–Mead)

After DE, Nelder–Mead further refines the solution on the full-resolution loss surface.

### 5. Stability check

The two-phase search is repeated with four random seeds (1, 7, 21, 99). Agreement across seeds confirms the solution is not an artifact of a single initialization.

### 6. Validation and submission

- Visual overlay of the fitted curve on the scatter plot.
- Mean nearest-neighbor distance reported as the internal fit-quality metric.
- Parameters rounded to four decimal places.
- Desmos parametric string generated for external verification.

---

## Final Results

### Recovered parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| **θ** | **0.5236 rad** | **30.0°** |
| **M** | **0.0300** | |
| **X** | **55.0001** | |

### Fit quality

| Metric | Value |
|--------|-------|
| Mean nearest-neighbor distance | **0.002053** |

The fitted parametric curve aligns closely with the scatter; the overlay plot in Step 9 of the notebook shows strong visual agreement.

### Stability across seeds

| Seed | θ | M | X | Loss |
|------|---|---|---|------|
| 1 | 0.523599 | 0.030000 | 55.000071 | 0.002053 |
| 7 | 0.523599 | 0.030000 | 55.000071 | 0.002053 |
| 21 | 0.523599 | 0.030000 | 55.000071 | 0.002053 |
| 99 | 0.523599 | 0.030000 | 55.000071 | 0.002053 |

Parameter spread across seeds is at numerical precision (Δθ ≈ 2×10⁻¹¹, ΔM ≈ 2×10⁻¹², ΔX ≈ 6×10⁻¹⁰), indicating a **stable, unique optimum** within the stated bounds.

### Desmos verification string

```
\left(t*\cos(0.5236)-e^{0.0300\left|t\right|}\cdot\sin(0.3t)\sin(0.5236)+55.0001,42+t*\sin(0.5236)+e^{0.0300\left|t\right|}\cdot\sin(0.3t)\cos(0.5236)\right)
```

Paste this into the [Desmos parametric calculator](https://www.desmos.com/calculator/rfj91yrxob) with domain **6 ≤ t ≤ 60** to visually confirm the fit.

---

## Observations

1. **Latent parameter `t`:** Nearest-neighbor point-to-curve distance is an effective objective when `t` is unobserved; it avoids incorrect point-to-point assignment.
2. **Two-phase optimization:** Coarse DE + full-resolution polish reduces runtime from 15+ minutes to under ~10 seconds per run while preserving accuracy on the full dataset.
3. **Convergence:** DE and Nelder–Mead agree; Step 7 refinement does not change the solution, indicating DE + polish already reached the local minimum.
4. **Robustness:** Identical results across four independent random seeds confirm the global minimum was found reliably within the bounded search space.
5. **Physical interpretation:** θ = 30° rotates the base curve; M = 0.03 introduces mild exponential growth in the oscillatory amplitude over |t|; X ≈ 55 shifts the curve horizontally.

---

## Conclusion

The unknown parameters of the parametric curve were successfully recovered from 1,500 `(x, y)` samples with no knowledge of per-point `t`:

- **θ = 0.5236 rad (30.0°)**
- **M = 0.0300**
- **X = 55.0001**

The mean point-to-curve distance of **0.002053** and the visual overlay demonstrate a high-quality fit. Multi-seed stability checks confirm the result is reproducible and trustworthy. The Desmos string above provides an independent verification path for submission.

---

## Project Files

| File | Description |
|------|-------------|
| `R&D_Task_Solved.ipynb` | Full analysis notebook (load → optimize → validate → Desmos string) |
| `xy_data.csv` | Input dataset (1,500 `(x, y)` points) |
| `README.md` | This document |

---

## How to Run

**Requirements:** Python 3.x, `numpy`, `pandas`, `matplotlib`, `scipy`

```bash
pip install numpy pandas matplotlib scipy
```

Place `xy_data.csv` in the same directory as the notebook, then run all cells in `R&D_Task_Solved.ipynb` sequentially. Expected total runtime is under a few minutes on a typical laptop.
