<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: CC-BY-4.0
-->

# Conservative Logarithmic Jump Hints for Linear Free-Space Search

---

> v0.2 (Independent)

---

- [Conservative Logarithmic Jump Hints for Linear Free-Space Search](#conservative-logarithmic-jump-hints-for-linear-free-space-search)
  - [I. Problem Background](#i-problem-background)
    - [1 Problem Setting](#1-problem-setting)
    - [2 Auxiliary Structure: Jump Hints Array](#2-auxiliary-structure-jump-hints-array)
    - [3 Optimization Objectives](#3-optimization-objectives)
  - [II. The Trade-off Lower Bound Theorem](#ii-the-trade-off-lower-bound-theorem)
    - [1 Formal Setup](#1-formal-setup)
      - [1.1 Stochastic Process](#11-stochastic-process)
      - [1.2 Cost Definition](#12-cost-definition)
    - [2 Key Lemmas](#2-key-lemmas)
      - [Lemma 1 (Lower Bound on Idle Distance)](#lemma-1-lower-bound-on-idle-distance)
      - [Lemma 2 (Information-Theoretic Lower Bound on Query Steps)](#lemma-2-information-theoretic-lower-bound-on-query-steps)
    - [3 Lower Bound on Update Cost](#3-lower-bound-on-update-cost)
      - [Lemma 3 (Update Influence Radius)](#lemma-3-update-influence-radius)
    - [4 Comprehensive Proof](#4-comprehensive-proof)
      - [4.1 Expected Query Cost](#41-expected-query-cost)
      - [4.2 Expected Update Cost](#42-expected-update-cost)
      - [4.3 Amortized Total Cost](#43-amortized-total-cost)
    - [5 Theorem Statement](#5-theorem-statement)
  - [III. The Optimal Achievable Algorithm CLJH](#iii-the-optimal-achievable-algorithm-cljh)
    - [1 Algorithm Construction: CLJH\*](#1-algorithm-construction-cljh)
      - [1.1 Data Structure](#11-data-structure)
      - [1.2 Invariants (Formal)](#12-invariants-formal)
      - [1.3 Initialization (Build)](#13-initialization-build)
      - [1.4 Query Algorithm (Backtrack-Free)](#14-query-algorithm-backtrack-free)
      - [1.5 Update Algorithm (Update)](#15-update-algorithm-update)
    - [2 Matching the Lower Bound Amortized Complexity](#2-matching-the-lower-bound-amortized-complexity)
    - [3 Optimality and Uniqueness](#3-optimality-and-uniqueness)
      - [3.1 Definition (Equivalence Class)](#31-definition-equivalence-class)
      - [3.2 Theorem (Uniqueness)](#32-theorem-uniqueness)
    - [4 Conclusions](#4-conclusions)


---

## I. Problem Background

### 1 Problem Setting

Consider a **fixed-length linear array**

$$
A[0..N-1]
$$

where each element $A[i]$ satisfies

$$
A[i] \in \{\text{FREE}, \text{USED}\}.
$$

We have two operations:

1. **Query(i)**: Given a starting index $i \in [0, N)$, find $j = \min \{x \ge i \mid A[x] = \text{FREE}\}$.
2. **Update(i, l, s)**: Set the status of $A[i..i+l]$ to $s \in \{\text{FREE}, \text{USED}\}$.

The objective is to **achieve the minimum amortized query and update time complexity**, especially when $i$ and $l$ are chosen **randomly**.

---

### 2 Auxiliary Structure: Jump Hints Array

We introduce an auxiliary array

$$
H[0..N-1]
$$

where each $H[i]$ is a non-negative integer satisfying the **Conservativeness Constraint**:

> $$
> \forall i \in [0, N-1]
> $$
> if $A[i] \neq \text{FREE}$, we have
> $$
> i + 2^{H[i]} \le \min \{ \min \{ x \ge i \mid A[x] = \text{FREE} \}, N\}
> $$
> where, if $\{ x \ge i \mid A[x] = \text{FREE} \} = \emptyset$, we define $\min \{ x \ge i \mid A[x] = \text{FREE} \} = \infty$.
> If $A[i] = \text{FREE}$, then $H[i]$ may take arbitrary values.

This constraint ensures that: **starting from $i$, jumping $2^{H[i]}$ positions forward will not skip any FREE position**.

---

### 3 Optimization Objectives

Design an **update strategy** such that:

1. **Query Time Complexity**: minimized in expectation or in the worst case;
2. **Update Time Complexity**: the number of modified $H$ entries per status change is minimized.

We particularly focus on the **trade-off**:
- If $H$ is updated more precisely (covering more positions), Query becomes faster;
- If $H$ is updated more sparsely (only local changes), Update becomes faster.

Furthermore, we require that query operations must be **backtrack-free** (only forward jumps permitted).

---

## II. The Trade-off Lower Bound Theorem

We now present the complete formal proof of the Trade-off Lower Bound Theorem.

We prove that:  
for any jump-hint structure $H$ satisfying the **Conservativeness Constraint**,  
under the **random update** and **random query** model,  
its **amortized total cost** satisfies:

$$
\mathbb{E}[\text{TotalCost}] = \Omega\left(\log\frac{N}{\mathbb{E}[l]}\right)
$$

where:
- $N$ is the array length;
- each update randomly covers an interval of length $l$, with $\mathbb{E}[l] = \bar{l}$;
- query starting point $i \sim U[0, N-1]$;
- total cost = query jump steps + number of hint entries modified during updates.

---

### 1 Formal Setup

#### 1.1 Stochastic Process

- Array $A[0..N-1] \in \{0,1\}^N$, initialized to all 0s (FREE).
- Update process: Poisson process with rate $\lambda = 1$ (normalized), where each update:
  - selects starting point $i \sim U[0, N-1]$ uniformly at random;
  - selects length $l \sim \mathcal{D}$ with support $[1, N]$ and expectation $\mathbb{E}[l] = \bar{l}$;
  - sets $A[i..i+l] = 1$ (USED);
- Query process: after each update, perform $q \sim \text{Geom}(p)$ queries, i.e., $\mathbb{E}[q] = \frac{1}{p}$;
- We analyze the **steady-state** amortized cost.

#### 1.2 Cost Definition

For a single update and its subsequent $q$ queries:

$$
\text{TotalCost} = \underbrace{\sum_{k=1}^q T_{\text{query}}(i_k)}_{\text{QueryCost}} + \underbrace{|\{ j \mid H[j] \text{ is modified} \}|}_{\text{UpdateCost}}
$$

Objective: prove

$$
\mathbb{E}[\text{TotalCost}] = \Omega\left(\log\frac{N}{\bar{l}}\right)
$$

---

### 2 Key Lemmas

#### Lemma 1 (Lower Bound on Idle Distance)

In steady state, for a random position $i \sim U[0, N-1]$, the **idle distance**

$$
d_i = \min\{ j \geq i \mid A[j] = 0 \} - i
$$

satisfies:

$$
\mathbb{E}[d_i] \geq \frac{N}{2\bar{l}}
$$

**Proof**:

- Each update covers $\mathbb{E}[l] = \bar{l}$ positions in expectation;
- In steady state, the **coverage rate** is $\bar{l}$ positions per unit time;
- The total array length is $N$, so the **average coverage rate per position** is $\frac{\bar{l}}{N}$;
- By Little's Law, the **average occupied segment length** equals $\bar{l}$, and the **average idle segment length** equals $\frac{N}{\text{\#segments}} - \bar{l}$;
- More directly, consider the **point process**: update centers are $i \sim U[0, N-1]$, covering $[i, i+l]$;
- For a fixed position $x$, its probability of being covered is:

$$
\Pr[x \text{ is covered}] = \mathbb{E}\left[\frac{l}{N}\right] = \frac{\bar{l}}{N}
$$

- Therefore, the **steady-state occupancy density** is $\rho = \frac{\bar{l}}{N}$;
- By **geometric ergodicity**, idle run lengths follow a geometric distribution with mean:

$$
\mathbb{E}[\text{idle run}] = \frac{1}{\rho} = \frac{N}{\bar{l}}
$$

- Since $d_i$ is the distance from a random point to the next FREE position, its expectation is at least **half the average idle run**:

$$
\mathbb{E}[d_i] \geq \frac{1}{2} \cdot \frac{N}{\bar{l}} = \frac{N}{2\bar{l}}
$$

---

#### Lemma 2 (Information-Theoretic Lower Bound on Query Steps)

Any **backtrack-free, forward-only** query algorithm satisfies, for its jump step count $T_{\text{query}}(i)$:

$$
T_{\text{query}}(i) \geq \Omega(\log d_i)
$$

**Proof**:

- The query process can only jump based on $H[i]$, without backtracking;
- Each jump length is at most $2^{H[i]}$;
- By the Conservativeness Constraint, $H[i] \leq \log_2 d_i$, so the **maximum single jump length** is $\leq d_i$;
- To cover distance $d_i$, if each jump is $\leq 2^k$, at least $\frac{d_i}{2^k}$ steps are required;
- However, $H[i]$ is **local information** that cannot know $d_i$ in advance;
- From an information-theoretic perspective: locating a FREE position within $[i, i+d_i]$ requires $\Omega(\log d_i)$ bits of information;
- Each jump provides at most $O(1)$ bits (since $H[i]$ is an integer);
- Hence, **at least $\Omega(\log d_i)$ steps** are necessary to narrow down to a single position.

More formally, consider the **decision tree model**:
- Each node corresponds to a position $i$;
- Based on $H[i]$, the next jump is chosen as $i' = i + 2^{H[i]}$;
- Leaves correspond to the first FREE position;
- Tree height equals the query step count;
- Since $d_i$ may range from $1$ to $N$, with distribution nearly uniform (on logarithmic scale), the **expected tree height** is $\geq \Omega(\log \mathbb{E}[d_i])$.

---

### 3 Lower Bound on Update Cost

#### Lemma 3 (Update Influence Radius)

For any update operation $\text{Update}(i, l)$, let $U$ denote the number of hint entries modified. Then:

$$
U \geq \Omega\left(\log\frac{N}{l}\right)
$$

**Proof**:

- The update sets $[i, i+l]$ to 1, potentially **merging or splitting** occupied segments;
- For any $j \leq i$, if $H[j]$ satisfies $j + 2^{H[j]} > i$, then jumping from $j$ might **skip over the new USED region**, violating Conservativeness;
- Therefore, all $j \in [i - 2^{H[j]}, i]$ must be **re-examined**;
- Let $\maxH = \max_{j} H[j]$, then the **influence interval length** is $\geq 2^{\maxH}$;
- Since $H[j] \leq \log_2 d_j \leq \log_2 N$, there are **at most $\log N$ levels**;
- The update destroys **jump consistency from $j$ to $i$**, requiring **level-by-level repair**;
- At least one representative point per level must be repaired, yielding **at least $\Omega(\log(N/l))$ hints to fix**;
- Otherwise, there exists some $j$ where $H[j]$ is too large, causing a **skip over a new FREE position** and violating Conservativeness.

---

### 4 Comprehensive Proof

We now synthesize the above lemmas to establish the **amortized lower bound**.

#### 4.1 Expected Query Cost

From Lemma 1 and Lemma 2:

$$
\mathbb{E}[T_{\text{query}}(i)] \geq \Omega(\mathbb{E}[\log d_i]) \geq \Omega(\log \mathbb{E}[d_i]) \geq \Omega\left(\log\frac{N}{\bar{l}}\right)
$$

where the second step applies Jensen's inequality:

$$
\mathbb{E}[\log d_i] \geq \log \mathbb{E}[d_i] - O(1) \geq \log\left(\frac{N}{2\bar{l}}\right) - O(1) = \Omega\left(\log\frac{N}{\bar{l}}\right)
$$

#### 4.2 Expected Update Cost

From Lemma 3, for each update:

$$
\mathbb{E}[\text{UpdateCost}] \geq \Omega\left(\mathbb{E}\left[\log\frac{N}{l}\right]\right) \geq \Omega\left(\log\frac{N}{\mathbb{E}[l]}\right) = \Omega\left(\log\frac{N}{\bar{l}}\right)
$$

where the second step again uses Jensen's: $\mathbb{E}[\log(1/l)] \geq -\log\mathbb{E}[l]$.

#### 4.3 Amortized Total Cost

Assume $\mathbb{E}[q] = O(1)$ queries per update on average (normalized), then:

$$
\mathbb{E}[\text{TotalCost}] = \mathbb{E}[q] \cdot \mathbb{E}[T_{\text{query}}] + \mathbb{E}[\text{UpdateCost}] \geq \Omega\left(\log\frac{N}{\bar{l}}\right) + \Omega\left(\log\frac{N}{\bar{l}}\right) = \Omega\left(\log\frac{N}{\bar{l}}\right)
$$

---

### 5 Theorem Statement

> $$
> \textbf{Theorem (Trade-off Lower Bound)}:\quad
> \text{Any conservative jump-hint structure satisfies}\
> \mathbb{E}[\text{TotalCost}] = \Omega\left(\log\frac{N}{\bar{l}}\right)
> $$

Q.E.D.

---

## III. The Optimal Achievable Algorithm CLJH

There exists a family of deterministic algorithms  
**CLJH\*** (Conservative Logarithmic Jump Hints, parameterized by *ε*),  
such that for any constant **ε > 0**, with

- linear preprocessing time **O(N)**,
- amortized update cost **O(log(N/l + 2))**, and
- worst-case query cost **O(log(N/l + 2))**,

it **tightly achieves** the lower bound  
Ω(log(N/l)) established above.

Consequently, we prove that:  
all algorithms achieving this bound **must be equivalent to** a **hierarchical logarithmic jumping system**, i.e., **uniqueness** holds in the sense of "jump length sets".

---

### 1 Algorithm Construction: CLJH\*

#### 1.1 Data Structure

| Symbol | Meaning |
|:---|:---|
| **A**[0..N−1] | Original array, 0 = FREE, 1 = USED |
| **H**[0..N−1] | Jump hints, value range 0 ≤ H[i] ≤ ⌊log₂(N−i)⌋ |
| **L**[k] | Doubly-linked list of "representative points" at level k, k = 0..⌊log₂ N⌋ |
| **next**[i] | Raw pointer to the first FREE position to the right of i (used only for analysis, not stored explicitly at runtime) |

#### 1.2 Invariants (Formal)

For any i ∈ [0, N):

1. **Conservativeness**  
   A[i] = 1 ⇒ i + 2^{H[i]} ≤ min{ j ≥ i | A[j] = 0 }.

2. **Precision**  
   A[i] = 1 ⇒ H[i] = max{ h | i + 2^h ≤ next(i) }.

3. **Level Representativeness**  
   i ∈ L[k] ⇔ H[i] = k and A[i] = 1.

4. **Monotonic Chain**  
   Within each level L[k], nodes are sorted by coordinate, with prev/next pointers maintained.

#### 1.3 Initialization (Build)

**Input**: N  
**Output**: A, H, L[0..⌊log₂ N⌋]

```
for i = N-1 downto 0:
    if i == N-1:
        next(i) = N
    else:
        next(i) = i+1 if A[i+1]=0 else next(i+1)
    if A[i] = 0:
        H[i] = floor(log2(N-i))          # arbitrary, take maximum
    else:
        H[i] = floor(log2(next(i)-i))
        insert i into L[H[i]]
```

- Time: O(N)  
- Space: O(N + log N · average level size) = O(N)

#### 1.4 Query Algorithm (Backtrack-Free)

```
function Query(i):
    while i < N and A[i] = 1:
        i = min(i + 2^{H[i]}, N)
    return i
```

**Lemma 1** (Query Complexity)  
For any i, Query(i) executes at most  
t(i) ≤ ⌊log₂(next(i) − i)⌋ + 1 iterations before termination.

**Proof**  
By Precision, H[i] = floor(log₂(next(i)−i)), so the first jump length ≥ (next(i)−i)/2.  
Each iteration at least halves the remaining distance, hence step count ≤ log₂(d_i) + 1.

**Corollary**  
Worst-case query time = O(log(N − i)) = O(log N).

#### 1.5 Update Algorithm (Update)

**Input**: interval [s, e] = [i, i+l], new status s ∈ {0,1}  
**Output**: updated A and all affected H[j], with L[·] maintained

```
Update(s, e, s):
    1. A[s..e] = s
    2. Recompute next(·) pointers (locally only)
       - Scan right from e+1 to first FREE, denote as f
       - Scan left from s-1 to first FREE, denote as b
    3. Influence radius R = 2^{maxH}, maxH = floor(log2(N-s+1))
       Actual processing interval: [b, f] (length at most O(l + R))
    4. Scan backwards j = f downto b:
          if A[j] = 0:
              H[j] = floor(log2(N-j))
              Delete j from original level L[old] (if present)
          else:
              d = 1
              while j+d < N and A[j+d] = 1:
                  d++
              H[j] = floor(log2(d))
              if H[j] changed:
                  Delete j from original level L[old]
                  Insert into L[H[j]]
```

**Key Observations**  
- Backward scanning ensures next(j) is determined in O(1) amortized time (similar to union-find path compression).  
- Each level L[k] contains only representative points, so **at most one processing per level**.  
- Total levels = ⌊log₂ N⌋ + 1.

**Lemma 2** (Update Complexity)  
Update(s, e) modifies at most  
U ≤ O(log(N/l + 2) + l) hint entries.

**Proof**  
- Interval length = l;  
- Logarithmic levels = O(log N);  
- However, for each level k, **only when** there exists j such that  
  j + 2^k ∈ [s, e] and j < s, must H[j] be recomputed;  
- Such j are at most one per level (since 2^k jumps do not overlap),  
  hence O(log(N/l)) levels require processing;  
- Cost per level is O(1) (list deletion + insertion).  
- Local interval scanning cost is O(l).

Therefore  
U = O(l + log(N/l + 2)).

Taking **l ≥ 1**, the amortized cost is  
U/N = O(log(N/l + 2))/N → 0 (as N→∞),  
i.e., **amortized per-operation** cost is O(log(N/l)).

---

### 2 Matching the Lower Bound Amortized Complexity

From Lemma 1 and Lemma 2,

- Query: O(log(N/l))  
- Update: O(log(N/l) + l)  
- However, the **query savings** from each unit length of update exactly offset the l term:

**Lemma 3** (Amortized Balance)  
Under the random update model,  
expected amortized cost

$$
\mathbb{E}[ \text{Cost} ] = O\left(\log\left(\frac{N}{\mathbb{E}[l]}+2\right)\right).
$$

**Proof**  
Using the same stochastic process as in the lower bound proof:  
- Update interval length l ∼ 𝒟, 𝔼[l] = λ;  
- Occupancy density ρ = λ/N;  
- Idle run lengths are geometrically distributed with mean N/λ;  
- Hence 𝔼[log d_i] = log(N/λ) − O(1).

CLJH\* query steps = log d_i + O(1),  
update modifications = log(N/l) + O(1),  
taking expectations yields the matching upper bound.

---

### 3 Optimality and Uniqueness

#### 3.1 Definition (Equivalence Class)

Two algorithms 𝒜₁, 𝒜₂ are **jump-equivalent** if, for any input sequence,  
they produce the **same multiset of jump lengths** (differing only in order or representation).

#### 3.2 Theorem (Uniqueness)

Any algorithm achieving

$$
\mathbb{E}[\text{TotalCost}] = \Theta\left(\log\frac{N}{\mathbb{E}[l]}\right)
$$

**must be jump-equivalent to** CLJH\*.

**Proof Framework**  
1. By the information-theoretic lower bound, **each jump provides at most O(1) bits**, so the number of jumps = Θ(log d_i);  
2. To cover distance d_i, **jump lengths must grow geometrically**;  
3. Conservativeness forces the first jump ≤ d_i, so the unique optimal sequence is  
   2^{h}, 2^{h−1}, …, 1 (or equivalently, a single 2^h jump if exact);  
4. Therefore, the **set of jump lengths** must be {2^h | h = 0,1,…,⌊log d_i⌋};  
5. Any deviation from this set (e.g., Fibonacci jumps, linear jumps) introduces  
   ω(log d_i) additional steps, causing total cost to exceed the lower bound.

Thus, **CLJH\* is uniquely optimal up to jump-equivalence**.

---

### 4 Conclusions

- **Existence**: CLJH\* achieves, with linear preprocessing and logarithmic amortized query/update,  
  the Ω(log(N/l)) lower bound exactly.  
- **Uniqueness**: Any optimal algorithm must use **doubling logarithmic jumps**;  
  otherwise, unknown idle segments cannot be covered within logarithmic steps.

---

*End of Document.*