# Closed-Form Solution for Optimal DEX Arbitrage

Exact analytic formulas for computing the profit-maximizing trade size in arbitrage across Uniswap V2-style constant product liquidity pools — from single-hop (2 pools) to arbitrary n-hop circular paths.

## Overview

When AMM pools along a trading path have price discrepancies, an arbitrage opportunity exists. The central problem is determining the **optimal input amount** that maximizes profit after fees.

The standard approach is **binary search**: iteratively narrow bounds on the input until the profit derivative crosses zero. This works but requires dozens of iterations and introduces approximation error.

This project derives **closed-form analytic solutions** that compute exact answers with zero iterations:

- **Single-hop (2 pools):** An $O(1)$ formula derived by direct differentiation of the profit function
- **Multi-hop (n pools):** An $O(n)$ formula using Möbius transformation composition, reducing the entire path to a single fractional-linear function

## Single-Hop Results

Using real SHIB/ETH reserve data from Uniswap V2 pools on Ethereum:

| Method | Optimal x | Profit | Time | Iterations |
|---|---|---|---|---|
| Binary Search | 7.921088 | 0.015955 | ~190 μs | ~53 |
| Closed-Form | 7.921088 | 0.015955 | ~145 μs | 0 |

## Multi-Hop Results

The general formula matches binary search across all tested path lengths:

| Path | Closed-Form x_opt | Profit | Binary Search Iterations |
|---|---|---|---|
| 2-hop | 7.921088 | 0.015955 | 54 |
| 3-hop | 19.078671 | 1.118604 | 50 |
| 4-hop | 85.827103 | 6.123027 | 50 |
| 5-hop | 162.151803 | 13.873792 | 50 |

The closed-form requires zero iterations for any number of hops. Analytic and simulated profits match to machine precision.

## The Formulas

### Single-Hop (2 Pools)

$$x_{\text{optimal}} = \frac{k\sqrt{abcd} - ad}{k(bk + d)}$$

where $a, b$ are the reserves of Pool A, $c, d$ are the reserves of Pool B, and $k = 1 - \text{fee}$ (e.g., $k = 0.997$ for the standard 0.3% Uniswap V2 fee).

### Multi-Hop (N Pools) — Möbius Transform

Each hop through a constant product AMM is a Möbius transformation. Composing $n$ hops via a 3-number recurrence produces:

$$l(x) = \frac{Kx}{M + Nx}$$

The optimal arbitrage input is:

$$x_{\text{opt}} = \frac{\sqrt{KM} - M}{N}$$

where $K$, $M$, $N$ are computed by iterating through the pools:

**Initialize** (first hop with reserves $r_1, s_1$ and fee $f_1$): $K = s_1 f_1$, $M = r_1$, $N = f_1$

**Update** for each subsequent hop: $K \leftarrow s_i f_i K$, $M \leftarrow r_i M$, $N \leftarrow r_i N + f_i K$

Arbitrage exists when $K/M > 1$.

## Setup

```bash
pip install -r requirements.txt
jupyter notebook
```

### Notebooks

- **`optimal_arbitrage.ipynb`** — Single-hop derivation and benchmarks
- **`multi_hop.ipynb`** — Multi-hop generalization via Möbius transforms, binary search comparison, and GPU batch analysis

### GPU Batch Analysis (Optional)

The multi-hop notebook includes a PyTorch section for evaluating thousands of candidate arbitrage paths in parallel on GPU. Requires:

```bash
pip install torch
```

On Apple Silicon Macs, the MPS backend provides native GPU acceleration.

## Medium Writeups

- [Part 1: Deriving a Closed-Form Solution for Optimal DEX Arbitrage](https://medium.com/@jeffreyhartigan/deriving-a-closed-form-solution-for-optimal-dex-arbitrage-b7c917bbe3e7)
- Part 2: Generalizing to N-Hop Paths via Möbius Transformations (coming soon)
