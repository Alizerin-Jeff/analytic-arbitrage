# Closed-Form Solution for Optimal DEX Arbitrage

Exact analytic formulas for computing the profit-maximizing trade size in arbitrage across Uniswap V2-style constant product liquidity pools, from single-hop (2 pools) to arbitrary n-hop circular paths.

An earlier writeup is here: [Geometry Of Arbitrage](https://github.com/Alizerin-Jeff/analytic-arbitrage/blob/main/Geometry_of_Arbitrage.pdf). It is being revised: swaps form a semigroup of linear fractional maps rather than a group, and the closed form itself has prior art (see [Prior Work](#prior-work)).

## Overview

When AMM pools along a trading path have price discrepancies, an arbitrage opportunity exists. The central problem is determining the **optimal input amount** that maximizes profit after fees.

A common approach is an **iterative search**: bisection or Newton's method on the profit derivative. This works but needs an iteration count, a tolerance, and a bracket or starting point.

This project derives **closed-form analytic solutions** that need none of those:

- **Single-hop (2 pools):** An $O(1)$ formula derived by direct differentiation of the profit function
- **Multi-hop (n pools):** An $O(n)$ formula. Each swap is a linear fractional (Möbius) map, so the entire path composes into a single map of the same form

## Single-Hop Results

Using real SHIB/ETH reserve data from Uniswap V2 pools on Ethereum (from `optimal_arbitrage.ipynb`):

| Method | Optimal x | Profit | Time | Iterations |
|---|---|---|---|---|
| Binary Search | 7.921090 | 0.015955 | ~190 μs | ~53 |
| Closed-Form | 7.921088 | 0.015955 | ~145 μs | 0 |

## Multi-Hop Results

From `multi_hop.ipynb`. Relative error is measured against a 50-digit reference.

| Path | Closed-Form x_opt | Profit | Closed-form rel. error | Bisection rel. error | Bisection iterations | Newton iterations |
|---|---|---|---|---|---|---|
| 2-hop | 7.921088 | 0.015955 | ~1e-14 | 1.5e-06 | 54 | 4 |
| 3-hop | 19.078671 | 1.118604 | ~1e-16 | 5.3e-08 | 50 | 5 |
| 4-hop | 85.827103 | 6.123027 | ~1e-15 | 2.2e-10 | 53 | 5 |
| 5-hop | 162.151803 | 13.873792 | ~1e-15 | 6.3e-09 | 54 | 5 |

The closed form requires zero iterations for any number of hops, and analytic and simulated profits match to machine precision. Bisection with a finite-difference derivative is less accurate in the optimal input because the profit curve is flat at its maximum; its profit is still essentially optimal.

On timing, the speedup depends on the baseline. In pure Python on one path the closed form is roughly 50 to 70 times faster than bisection and roughly 4 to 14 times faster than Newton's method with exact derivatives. Exact figures vary by machine; see the notebook.

## The Formulas

### Single-Hop (2 Pools)

$$x_{\text{optimal}} = \frac{k\sqrt{abcd} - ad}{k(bk + d)}$$

where $a, b$ are the reserves of Pool A, $c, d$ are the reserves of Pool B, and $k = 1 - \text{fee}$ (e.g., $k = 0.997$ for the standard 0.3% Uniswap V2 fee).

### Multi-Hop (N Pools)

Each hop through a constant product AMM is a linear fractional map fixing the origin. Composing $n$ hops via a 3-number recurrence produces:

$$l(x) = \frac{Kx}{M + Nx}$$

The optimal arbitrage input is:

$$x_{\text{opt}} = \frac{\sqrt{KM} - M}{N}$$

where $K$, $M$, $N$ are computed by iterating through the pools:

**Initialize** (first hop with reserves $r_1, s_1$ and fee $f_1$): $K = s_1 f_1$, $M = r_1$, $N = f_1$

**Update** for each subsequent hop: $K \leftarrow s_i f_i K$, $M \leftarrow r_i M$, $N \leftarrow r_i N + f_i K$ (using the value of $K$ from before the update)

Arbitrage exists when $K/M > 1$.

### Equivalent Two-Scalar Form

In the reciprocal coordinate $u = 1/x$ a swap is the affine map $u \mapsto \alpha u + \beta$ with $\alpha = r/(fs)$ and $\beta = 1/s$. Composition needs two numbers:

**Initialize**: $A = \alpha_1$, $B = \beta_1$

**Update**: $A \leftarrow \alpha_i A$, $B \leftarrow \alpha_i B + \beta_i$

Then $l(x) = \dfrac{x}{A + Bx}$, with $A = M/K$ and $B = N/K$, and

$$x_{\text{opt}} = \frac{\sqrt{A} - A}{B}, \qquad \text{profit}_{\max} = \frac{(1 - \sqrt{A})^2}{B}$$

Arbitrage exists when $A < 1$.

**Which to use.** $K$ and $M$ are products of raw reserves and overflow floating point on long paths (float32 after about five hops with reserves near $10^7$). $A$ is a product of exchange-rate ratios and stays near 1 around a cycle, so $(A, B)$ is the stable choice in floating point. $(K, M, N)$ has no division and suits exact integer arithmetic.

## Setup

```bash
pip install -r requirements.txt
jupyter notebook
```

### Notebooks

- **`optimal_arbitrage.ipynb`** - Single-hop derivation and benchmarks
- **`multi_hop.ipynb`** - Multi-hop generalization, comparison against bisection and Newton's method, and batch evaluation on CPU and GPU

### Batch Evaluation (Optional)

The multi-hop notebook includes a PyTorch section for evaluating many candidate cycles in parallel. Requires:

```bash
pip install torch
```

The batch section uses the $(A, B)$ recurrence, compares CPU and GPU at matched float32 precision, and includes a batched Newton baseline on the same device. On Apple Silicon Macs the MPS backend provides GPU acceleration (float32 only).

## Prior Work

The reduction of a cycle of constant-product pools to a single equivalent pool, and the resulting closed-form optimal input, appear in:

- Wang, Chen, Wu, Zhou, Deng, Wattenhofer. "Cyclic Arbitrage in Decentralized Exchanges." [arXiv:2105.02784](https://arxiv.org/abs/2105.02784) (2021).

What this repository adds is the structural view (swaps as linear fractional maps, affine in the reciprocal coordinate) and the recurrences that follow from it.

## Medium Writeups

- [Part 1: Deriving a Closed-Form Solution for Optimal DEX Arbitrage](https://medium.com/@jeffreyhartigan/deriving-a-closed-form-solution-for-optimal-dex-arbitrage-b7c917bbe3e7)
- [Part 2: Generalizing to N-Hop Paths via Möbius Transformations](https://medium.com/@jeffreyhartigan/the-geometry-of-arbitrage-generalizing-multi-hop-dex-paths-via-möbius-transformations-b56f992b01cd)
