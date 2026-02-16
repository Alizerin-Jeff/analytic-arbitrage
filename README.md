# Closed-Form Solution for Optimal DEX Arbitrage

An exact analytic formula for computing the profit-maximizing trade size in single-hop arbitrage between two Uniswap V2-style liquidity pools.

## Overview

When two AMM pools for the same token pair have different prices, an arbitrage opportunity exists: buy from the cheaper pool, sell to the more expensive one, and pocket the difference. The central problem is determining the **optimal input amount** that maximizes profit after fees.

The standard approach is **binary search**. Iteratively narrow bounds on the input until the profit derivative crosses zero. This works but requires dozens of iterations and introduces approximation error.

This project derives a **closed-form analytic solution** by expressing profit as a function of the input amount, differentiating, and solving for the root directly. The result is an exact answer computed in O(1) time with zero iterations.

## Results

Using real SHIB/ETH reserve data from Uniswap V2 pools on Ethereum:

| Method | Optimal x | Profit | Time | Iterations |
|---|---|---|---|---|
| Binary Search | 7.921088 | 0.015955 | ~200 μs | ~40 |
| Closed-Form | 7.921088 | 0.015955 | ~3 μs | 0 |

Both methods converge to the same answer. The closed-form solution is orders of magnitude faster and deterministic which is important for gas-sensitive on-chain contexts.

## The Formula

$$x_{\text{optimal}} = \frac{k\sqrt{abcd} - ad}{k(bk + d)}$$

where $a, b$ are the reserves of Pool A, $c, d$ are the reserves of Pool B, and $k = 1 - \text{fee}$ (e.g., $k = 0.997$ for the standard 0.3% Uniswap V2 fee).

Derived by setting $\frac{\partial}{\partial x}[\text{profit}(x)] = 0$ and solving for $x$, where profit is computed through the constant product AMM equations with fee adjustments at each hop.

## Setup

```bash
pip install -r requirements.txt
jupyter notebook optimal_arbitrage.ipynb
```

## Medium Writeup

[*[Optimal DEX Arbitrage]*](https://medium.com/@jeffreyhartigan/deriving-a-closed-form-solution-for-optimal-dex-arbitrage-b7c917bbe3e7)
