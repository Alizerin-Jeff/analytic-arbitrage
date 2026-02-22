# Generalizing Optimal DEX Arbitrage to N-Hop Paths with Möbius Transformations

## Introduction

In a [previous article](https://medium.com/@jeffreyhartigan/deriving-a-closed-form-solution-for-optimal-dex-arbitrage-b7c917bbe3e7), I derived a closed-form formula for the optimal trade size in single-hop arbitrage between two Uniswap V2-style liquidity pools. The formula replaced iterative binary search with a single algebraic expression:

$$x_{\text{ideal}} = \frac{k\sqrt{abcd} - ad}{k(bk + d)}$$

I noted at the end that this approach "does not generalize" easily to multi-hop paths because "multi-hop arbitrage paths involving three or more pools produce profit functions with higher-degree polynomials under the square root." That turned out to be wrong.

The algebra has a hidden structure. Every constant product AMM swap is a Möbius transformation, and Möbius transformations compose cleanly via matrix multiplication. This means the entire n-hop path — regardless of how many pools are involved — collapses to the exact same functional form as a single hop. The closed-form solution generalizes perfectly.

## Recap: The Single-Hop Formula

The setup: two constant product pools for the same token pair, Pool A with reserves $(a, b)$ and Pool B with reserves $(c, d)$. Borrow $x$ of token₀, swap through Pool A to get token₁, swap that through Pool B to get token₀ back, keep the difference. The fee factor $k = 0.997$ (for Uniswap V2's 0.3% fee) is applied to each swap input.

The key equations are the constant product swap outputs:

$$y = b - \frac{ab}{a + kx}, \quad z = c - \frac{cd}{d + ky}$$

Profit is $z - x$. I found the optimal input by setting $\frac{dz}{dx} = 1$ and solving, which produced the formula above.

The natural question: what happens with three pools? Four? Ten? Can we still avoid binary search?

## The Key Insight: AMM Swaps Are Möbius Transformations

Start with the single-hop swap output. The formula $y = b - \frac{ab}{a + kx}$ simplifies algebraically:

$$y = \frac{b(a + kx) - ab}{a + kx} = \frac{bkx}{a + kx}$$

This has the form $\frac{\alpha x}{\beta + \gamma x}$, which is a Möbius transformation (also called a linear fractional transformation). More specifically, it is a Möbius transformation that maps zero to zero: if you swap zero tokens in, you get zero out.

Möbius transformations have a remarkable property: composing any two of them yields another Möbius transformation. And when both transformations fix the origin (send $0 \to 0$), so does their composition. This means chaining any number of constant product swaps always produces a function of the form:

$$l(x) = \frac{Kx}{M + Nx}$$

where $K$, $M$, $N$ are constants built from the pool reserves and fees. The function never gets more complicated than this, no matter how many hops.

This is why my original concern about "higher-degree polynomials" was misplaced. The nested fractions that appear when you substitute each hop into the next look complicated, but they always simplify to the same two-term ratio. The Möbius structure guarantees it.

## Matrix Representation

Each hop can be represented as a $2 \times 2$ matrix. A swap through a pool with input reserve $r$, output reserve $s$, and fee factor $f$ corresponds to:

$$\mathbf{M}_i = \begin{pmatrix} s_i f_i & 0 \\ f_i & r_i \end{pmatrix}$$

These matrices act on the variable $x$ through the rule: if $\mathbf{M} = \begin{pmatrix} a & b \\ c & d \end{pmatrix}$, then the associated function is $f(x) = \frac{ax + b}{cx + d}$.

Composing $n$ hops is matrix multiplication:

$$\mathbf{M}_{\text{total}} = \mathbf{M}_n \cdot \mathbf{M}_{n-1} \cdots \mathbf{M}_1 = \begin{pmatrix} K & 0 \\ N & M \end{pmatrix}$$

The top-right entry stays exactly zero throughout the multiplication, because every individual matrix has a zero there (reflecting the $0 \to 0$ property). The result reads off directly: $l(x) = \frac{Kx}{M + Nx}$.

## The 3-Number Recurrence

In practice, you do not need to multiply matrices. Maintaining three scalars $K$, $M$, $N$ and updating them per hop is sufficient and numerically cleaner.

Initialize with the first hop (reserves $r_1, s_1$, fee $f_1$): $K = s_1 f_1$, $M = r_1$, $N = f_1$.

For each subsequent hop $i$ with reserves $r_i, s_i$ and fee $f_i$, update simultaneously:

$$K \leftarrow s_i f_i K, \quad M \leftarrow r_i M, \quad N \leftarrow r_i N + f_i K$$

The update for $N$ uses the value of $K$ before it is updated in this step. After processing all hops, you have the final $K$, $M$, $N$.

This is $O(n)$ — one pass through the pools, three multiplications and one addition per hop.

## The General Optimal Formula

With $l(x) = \frac{Kx}{M + Nx}$, the derivative is:

$$\frac{dl}{dx} = \frac{KM}{(M + Nx)^2}$$

Setting $\frac{dl}{dx} - 1 = 0$ (the same marginal-cost-equals-marginal-revenue condition as the single-hop case):

$$(M + Nx)^2 = KM$$

$$x_{\text{opt}} = \frac{\sqrt{KM} - M}{N}$$

This works for any number of hops. Profitable arbitrage exists when $K/M > 1$, meaning the initial marginal exchange rate through the full path exceeds unity.

## Verification: The 2-Hop Case

To confirm the general formula subsumes the original, map the 2-pool setup into the recurrence. Hop 1 uses Pool A ($r = a$, $s = b$, $f = k$). Hop 2 uses Pool B ($r = d$, $s = c$, $f = k$). Running the recurrence:

After hop 1: $K = bk$, $M = a$, $N = k$.

After hop 2: $K = bck^2$, $M = ad$, $N = k(bk + d)$.

Substituting:

$$x_{\text{opt}} = \frac{\sqrt{bck^2 \cdot ad} - ad}{k(bk + d)} = \frac{k\sqrt{abcd} - ad}{k(bk + d)}$$

Exactly the original formula. The generalization reduces to the special case as expected.

Using the same SHIB/ETH reserve data from the first article, the multi-hop recurrence reproduces the original answer to machine precision ($\sim 10^{-16}$ difference).

## Multi-Hop Benchmarks

I tested the general formula on synthetic circular arbitrage paths of varying length, each with carefully chosen reserves to create realistic price discrepancies. For every path, I verified the analytic result against both a direct simulation (feeding $x_{\text{opt}}$ through the actual swap chain) and binary search.

| Path | Closed-Form x_opt | Profit | Binary Search Iterations |
|---|---|---|---|
| 2-hop | 7.921088 | 0.015955 | 54 |
| 3-hop | 19.078671 | 1.118604 | 50 |
| 4-hop | 85.827103 | 6.123027 | 50 |
| 5-hop | 162.151803 | 13.873792 | 50 |

The closed-form and binary search agree on the optimal input to at least 6 decimal places in every case. The closed-form requires zero iterations regardless of path length.

## Complexity and Performance

The single-hop formula was $O(1)$: a fixed number of arithmetic operations regardless of input. The multi-hop formula is $O(n)$: one pass through $n$ pools to compute $K$, $M$, $N$, then a constant-time square root. This is the theoretical minimum — you must read the reserves of each pool at least once.

Binary search is $O(n \cdot I)$, where $I$ is the number of iterations (typically 40-60). Each iteration evaluates the full multi-hop simulation, which itself costs $O(n)$. So binary search scales quadratically in effect, while the closed-form scales linearly.

In benchmarks from 2 to 20 hops, the closed-form maintains a consistent speedup over binary search, with the gap widening as path length increases.

## GPU-Accelerated Batch Evaluation

In production MEV systems, the bottleneck is not evaluating a single path. It is scanning thousands of candidate paths simultaneously. An arbitrage bot enumerating routes across a DEX graph might evaluate 10,000+ candidate paths per block.

Because the Möbius recurrence operates on independent scalar arithmetic per path, the computation is embarrassingly parallel. Each candidate path runs its own 3-number recurrence with no cross-path dependencies. This maps directly to GPU hardware.

The GPU parallelism works by promoting the three scalars $K$, $M$, $N$ from scalars to tensors of shape `(batch_size,)` — one entry per candidate route. The recurrence loop still iterates sequentially over hops (typically 3–5), but each iteration performs a single element-wise operation that updates all routes simultaneously. There is no inner loop over paths; PyTorch broadcasts the arithmetic across the entire batch in one vectorized call per hop.

Concretely: for a batch of 10,000 five-hop routes, the loop runs exactly five iterations. Each iteration updates all 10,000 values of $K$, $M$, and $N$ in parallel. After five iterations, you have the optimal input and profit for every route in one pass. This is the actual mechanism of GPU acceleration — vectorized scalar recurrence over the batch dimension, not literal $2\times2$ matrix multiplications per path.

On Apple Silicon Macs, the MPS (Metal Performance Shaders) backend provides native GPU acceleration. On NVIDIA hardware, CUDA applies. The implementation is a few dozen lines of tensor operations.

For large batch sizes (10,000+ paths), GPU evaluation provides significant speedup over sequential CPU computation. The exact ratio depends on hardware, but the structure of the problem — uniform computation per path, no branching, no inter-path communication — is ideal for GPU architectures.

## Practical Considerations

The same limitations from the original analysis apply and compound in the multi-hop setting. The formula assumes constant product AMMs (Uniswap V2 style). It does not apply to concentrated liquidity (V3), stableswap curves, or other invariants.

In multi-hop paths, each pool may have a different fee. The recurrence handles this naturally — each hop carries its own fee factor $f_i$. Mixed-fee paths work without modification.

The formula also assumes reserves are static during execution. In a multi-hop atomic transaction (e.g., via flashloan), this holds by construction. But if the path spans multiple transactions or if another trader front-runs part of the path, the reserves change and the formula's output is stale. This is the standard MEV race condition, not specific to the closed-form approach.

Numerical stability deserves mention. The product $KM$ can become very large for long paths (it is the product of all output reserves times all input reserves times all fees). For paths beyond ~20 hops with large reserves, working in log-space or using arbitrary precision may be necessary. In practice, economically meaningful arbitrage paths rarely exceed 5-6 hops on current DEX graphs.

## Conclusion

The original closed-form solution for two-pool arbitrage was a special case of a much cleaner general structure. Every constant product AMM swap is a Möbius transformation. Möbius transformations compose via matrix multiplication. The entire n-hop path reduces to three numbers and a square root:

$$x_{\text{opt}} = \frac{\sqrt{KM} - M}{N}$$

The formula produces exact results with zero iteration for any number of hops. It is the natural generalization of the single-hop result, and it was hiding in plain sight in the algebra of constant product invariants.

The code and full derivation are available on [GitHub](https://github.com/Alizerin-Jeff/analytic-arbitrage).
