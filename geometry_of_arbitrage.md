# The Geometry of Arbitrage: Generalizing Multi-Hop DEX Paths via Möbius Transformations

In my previous post, I derived a closed-form solution for optimal arbitrage between two constant product liquidity pools. It replaced the standard iterative binary search with a single algebraic expression:

$$x_{\text{ideal}} = \frac{k\sqrt{abcd} - ad}{k(bk + d)}$$

At the time, I assumed this was a special case. I wrote that multi-hop paths, the circular trades going through 3, 4, or more pools, would likely produce "higher-degree polynomials" that didn't admit clean analytic solutions.

I was wrong.

It turns out that the algebra of constant product AMMs has a hidden, elegant structure. If you look at it the right way, arbitrage isn't just calculus, it's geometry. And the solution for a 100-hop path is just as clean as the solution for a single hop.

It all started with a literal back-of-the-envelope calculation.

### The Envelope Insight

I was doodling on the back of an envelope, manually working out the input-output function for a 4-hop route. I wasn't trying to solve for $x$ yet; I just wanted to see the shape of the function. I noticed an interesting continued fraction structure forming.  

As I chained the swaps together, I noticed the fractions weren't getting messier. They were simplifying. No matter how many pools I added, the resulting transfer function always collapsed into the same form:

$$l(x) = \frac{Kx}{M + Nx}$$

To a physicist or mathematician, this shape is instantly recognizable. It is a Möbius transformation. Specifically, one that fixes the origin (zero tokens in, zero tokens out).

Möbius transformations are fundamental to complex analysis and projective geometry. They are the automorphisms of the Riemann sphere. And they have a powerful property: **they form a group under composition.** If Swap A is a Möbius transformation and Swap B is a Möbius transformation, then the path A $\to$ B is also just a single Möbius transformation.

You don't need to simulate the trade hop-by-hop. You can mathematically collapse the entire route, whether 3 hops or 100 hops, into a single $2 \times 2$ matrix.

---

*(Note: If you are just here for the performance benchmarks and GPU acceleration results, feel free to skip down to the Performance section. But if you want to see the engine, let's look at the math.)*

### The Math: AMMs as Matrices

A standard constant product swap with input reserve $r$, output reserve $s$, and fee factor $f$ maps an input $x$ to an output via:

$$\text{output} = \frac{s \cdot f \cdot x}{r + f \cdot x}$$

This is algebraically identical to the familiar formula from the single-hop derivation:

$$y = b - \frac{ab}{a + kx} = \frac{b(a + kx) - ab}{a + kx} = \frac{bkx}{a + kx}$$

Both have the form $\frac{\alpha x}{\beta + \gamma x}$, a linear fractional transformation that fixes the origin. We can represent each swap as a $2 \times 2$ matrix:

$$\mathbf{M}_i = \begin{pmatrix} s_i f_i & 0 \\ f_i & r_i \end{pmatrix}$$

where the matrix acts on $x$ through the standard rule: 

$$\text{ if  } \mathbf{M} = \begin{pmatrix} a & b \\ c & d \end{pmatrix}, \text{ then } f(x) = \frac{ax + b}{cx + d}$$

To calculate a multi-hop route, you simply multiply the matrices:

$$\prod_{i=0}^{n-1} \mathbf{M}_{n-i} = \mathbf{M}_n \cdot \mathbf{M}_{n-1} \cdots \mathbf{M}_1 = \mathbf{M}_{\text{total}}= \begin{pmatrix} K & 0 \\ N & M \end{pmatrix}$$

The top-right entry stays exactly zero throughout the multiplication, a consequence of every individual map fixing the origin. The resulting matrix gives us three coefficients $K$, $M$, and $N$, and the entire path, no matter how many hops, reduces to:

$$l(x) = \frac{Kx}{M + Nx}$$

### The 3-Number Recurrence

In practice, you don't need to multiply full matrices. Maintaining three scalars and updating them per hop is sufficient and numerically cleaner.

**Initialize** with the first hop (reserves $r_1, s_1$, fee $f_1$):
$$K = s_1 f_1, \quad M = r_1, \quad N = f_1$$

**Update** for each subsequent hop $i$ with reserves $r_i, s_i$ and fee $f_i$:
$$K \leftarrow s_i f_i K, \quad M \leftarrow r_i M, \quad N \leftarrow r_i N + f_i K$$

*(Note: The update for $N$ uses the value of $K$ before it is updated in this step.)*

This is a direct consequence of the matrix multiplication:

$$ \begin{pmatrix} s_i f_i & 0 \\ f_i & r_i \end{pmatrix} \begin{pmatrix} K & 0 \\ N & M \end{pmatrix} =\begin{pmatrix} s_i f_i K & 0 \\ r_i N + f_i K & r_i M \end{pmatrix}  $$

After processing all hops, you have the final $K$, $M$, $N$. This is $O(n)$ with one pass through the pools, three multiplications, and one addition per hop.

### The General Solution

Once we have the path reduced to this simple rational function, finding the optimal input is straightforward. Take the derivative of the profit function $l(x) - x$, and set it to zero:

$$\frac{dl}{dx} = \frac{KM}{(M + Nx)^2} = 1$$

$$(M + Nx)^2 = KM$$

$$\boxed{x_{\text{opt}} = \frac{\sqrt{KM} - M}{N}}$$

This formula requires zero iterations. It is exact. And it scales linearly $O(n)$ with the number of hops, which is the theoretical minimum, since you must read the reserves of each pool at least once.

### The Free Profitability Check

There's a highly practical bonus hiding in these same three numbers. For $x_{\text{opt}}$ to be positive, the numerator $\sqrt{KM} - M$ must be positive, which requires $K > M$. Equivalently:

$$\frac{K}{M} > 1 \implies \text{profitable arbitrage exists}$$

This ratio is the initial marginal exchange rate of the entire path and the instantaneous rate you'd get on an infinitesimally small trade. If it exceeds 1, there's money on the table. If it doesn't, no amount of optimization will help.

You get this check for free from the same $O(n)$ recurrence that computes the optimal trade size. No simulation needed. In a production system scanning thousands of candidate routes, this means you can filter for profitability and compute optimal amounts in a single pass.

---

### Verification & Performance

> **Recovering the 2-Hop Formula:** If we run the recurrence on a standard 2-pool setup (Pool A: $r=a, s=b$, Pool B: $r=d, s=c$), the final coefficients become $K = bck^2$, $M = ad$, and $N = k(bk + d)$. Plugging these into our general solution yields exactly the original single-hop formula: $\frac{k\sqrt{abcd} - ad}{k(bk + d)}$.

I tested the general formula on synthetic circular arbitrage paths of varying length. For every path, the analytic result was verified against both a direct simulation and a traditional binary search. 

| Path | Closed-Form $x_{\text{opt}}$ | Profit | Binary Search Iterations |
| :--- | :--- | :--- | :--- |
| **2-hop** | 7.921088 | 0.015955 | 54 |
| **3-hop** | 19.078671 | 1.118604 | 50 |
| **4-hop** | 85.827103 | 6.123027 | 50 |
| **5-hop** | 162.151803 | 13.873792 | 50 |

The closed-form and binary search agree to at least 6 decimal places in every case. The closed-form requires zero iterations regardless of path length.

### From CPU to GPU (The Parallel Unlock)

Because we aren't iterating, the closed-form solution on a CPU is roughly **50–60x faster** than binary search across all tested path lengths. Binary search scales as $O(n \cdot I)$ where $I \approx 50$ iterations, whereas the closed-form scales strictly as $O(n)$.

But the real unlock is parallelism. In a production MEV system, you aren't checking one route but scanning thousands.

Because the Möbius recurrence is just arithmetic on three independent scalars per path, it maps trivially to GPU tensor operations. Promote $K$, $M$, $N$ from scalars to tensors of shape `(batch_size,)`, and the entire batch updates in a single vectorized operation per hop. No cross-path dependencies. No branching. Embarrassingly parallel.

* At roughly **10,000 parallel routes**, the GPU overtakes the CPU (overcoming dispatch overhead).
* At **1,000,000 routes**, the GPU is over **5x faster**, evaluating the entire search space in roughly 50 milliseconds.

The strategic picture: you can now scan every candidate route in a single batched computation, get the profitability check ($K/M > 1$), the optimal trade size, and the expected profit for every path simultaneously. Filter, sort, execute.

### The Physics Connection

This is where it gets weirdly beautiful.

Möbius transformations are not some niche algebraic trick. They are the symmetry group of the extended complex plane - the conformal maps that preserve circles and angles. They appear in complex analysis, hyperbolic geometry, and the theory of continued fractions. In physics, they are isomorphic to the Lorentz group: the mathematical structure that describes how space and time transform as you approach the speed of light.

When we model multi-hop arbitrage this way, we are mapping financial liquidity onto the geometry of the complex plane. The $2 \times 2$ matrices we multiply to combine swap hops are the same mathematical objects that describe how the Riemann sphere maps onto itself, how relativistic velocities compose, and how electrical engineers chain impedance networks.

Nobody designed $xy = k$ with Möbius transformations in mind. The constant product invariant was chosen for its simplicity as a minimal viable AMM. The geometric structure was just sitting there, latent in the algebra, waiting to be noticed.

Tell me we live in a simulation without telling me we live in a simulation.

### Practical Considerations

* **AMM Types:** The formula assumes constant product AMMs. It does not apply to concentrated liquidity or stableswap curves.
* **Fees:** Multi-hop paths may have different fees per pool. The recurrence handles this naturally; each hop carries its own $f_i$. 
* **State Freshness:** The formula assumes reserves are static during execution. If another trader front-runs part of the path, the output is stale and the standard MEV race condition applies.
* **Overflow:** For very long paths with massive reserves, the product $KM$ can overflow standard floating point, something I found early on in testing.

### Conclusion

The original closed-form solution for two-pool arbitrage was a special case of a much cleaner general structure. Every constant product AMM swap is a Möbius transformation. Möbius transformations compose via matrix multiplication. The entire n-hop path reduces to three numbers and a square root:

$$x_{\text{opt}} = \frac{\sqrt{KM} - M}{N}$$

Zero iteration. Exact results. Any number of hops. The code, the derivation, and the benchmarks are available on [GitHub](https://github.com/Alizerin-Jeff/analytic-arbitrage).

Sometimes infrastructure hides elegant mathematics.
And sometimes you find it on the back of an envelope.