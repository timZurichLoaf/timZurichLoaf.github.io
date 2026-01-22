---
layout: post
title: "[invisible] Chronology of Distributed Semi-Joins"
---

## Probabilistic Filter for Membership Queries (1970) 
[Bloom Filter](https://dl.acm.org/doi/10.1145/362686.362692) is a space-efficient probabilistic data structure for fast membership queries.

Two key components of a Bloom filter are bit array and hash functions.
Say we start with as bit array of size $10$ and $2$ hash functions,

$$h_1(x) = x \bmod 10; h_2(x) = (x + 4) \bmod 10.$$

Initially, all the bits are set to $0$, meaning no element has been inserted yet.

{:class="table table-bordered table-hover table-sm"}
| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

To insert `1`,
because $h_1(1) = 1$ and $h_2(1) = 5$, 
the bits at positions **1** and **5** are set to $1$.

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | <span style="color:red">1</span> | 0 | 0 | 0 | <span style="color:red">1</span> | 0 | 0 | 0 | 0 |

### Insert `3`

Because $h_1(3) = 3$ and $h_2(3) = 7$, 
the bits at positions **3** and **7** are set to $1$.

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 0 |

### Query `3` (true positive)

$3$ has been inserted, as the bits at positions 3 and 7 are **both** $1$.


### Query `5` (true negative)

$5$ has NOT been inserted, as the bit at position $h_1(5) = 5$ is $1$ but that at $h_2(5) = 9$ is $\color{blue}0$.

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 1 | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 1 | 0 | <span style="color:blue">0</span> |

### Query `7` (<span style="color:red">false positive</span>)

Now we got the wrong answer. Though it is never
inserted, 
the bits at positions $h_1(7) = 7$ and $h_2(7) = 1$ are both $1$.

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | <span style="color:red">1</span> | 0 | 1 | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 0 |

### Trade Accuracy for Efficiency

The time & space efficiency of Bloom Filter comes at the cost of accuracy. 
Hash collisions may result in <span style="color:red">false positives</span>.


## Exact Semi-joins (1981)

A semi-join is a batch of membership queries. Say we have two single-column tables $R(A)$ and $S(A)$.

<table style="display:inline-block; margin-right:40px;vertical-align:top; text-align:center;">
  <!-- <caption><b>R(A)</b></caption> -->
  <tr><th>R(A)</th></tr>
  <tr><td>1</td></tr>
  <tr><td><span style="color:red">2</span></td></tr>
  <tr><td>3</td></tr>
  <tr><td><span style="color:red">4</span></td></tr>
</table>
<!--  -->
<table style="display:inline-block; vertical-align:top;  text-align:center;">
  <!-- <caption><b>S(A)</b></caption> -->
  <tr><th>S(A)</th></tr>
  <tr><td>2</td></tr>
  <tr><td>4</td></tr>
</table>

<!-- --- -->

The semi-join $R \ltimes S$ keeps only $\color{red}{2, 4}$ in **R** who also appear in **S**.

[Yannakakis Algorithm](https://dl.acm.org/doi/10.5555/1286831.1286840) 
uses semi-joins to filter out **every** irrelevant tuple,
so that the join is executed later on the pruned tables with many fewer tuples.

For example, we want to join the following three table $R(A, B) \bowtie S(B, C) \bowtie T(C)$.

<table style="display:inline-block; margin-right:40px; vertical-align:top; text-align:center;">
  <thead>
    <tr>
      <th colspan="2">R(A, B)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td><span style="color:red">1</span></td><td><span style="color:red">1</span></td></tr>
    <tr><td><span style="color:red">2</span></td><td><span style="color:red">1</span></td></tr>
    <tr><td>3</td><td>2</td></tr>
    <tr><td>4</td><td>2</td></tr>
  </tbody>
</table>
<!--  -->
<table style="display:inline-block; margin-right:40px; vertical-align:top; text-align:center;">
  <thead>
    <tr>
      <th colspan="2">S(B, C)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>1</td></tr>
    <tr><td>1</td><td>2</td></tr>
    <tr><td><span style="color:red">1</span></td><td><span style="color:red">3</span></td></tr>
    <tr><td>1</td><td>4</td></tr>
  </tbody>
</table>
<!--  -->
<table style="display:inline-block; vertical-align:top; text-align:center;">
  <tr><th>T(C)</th></tr>
  <tr><td><span style="color:red">3</span></td></tr>
</table>


The semi-join $S \ltimes T$ finds it enough to consider the tuple $(1, 3)$ in $S(B, C)$.
The other semi-join $R \ltimes S$ finds $(1, 1)$ and $(2, 1)$ in $R(A, B)$.
After pruning, we only need to consider the highlighted <span style="color:red">tuples</span> while executing the join.


Now the questions are
1. How to do a semi-join better (faster & more compactly)?
2. In what order to perform those semi-joins?


## Distributed Semi-join with Filter (1990)

The very purpose of semi-joins is to prune the tables.
If we don't need to filter out **every** irrelevant tuple, 
[J.K. Mullin](https://ieeexplore.ieee.org/document/52778) finds [Bloom Filter](https://dl.acm.org/doi/10.1145/362686.362692) an answer to the first question. It is easy to compute and compact, so cheap to transmit over a network. It only gives **false positives** when a membership is queried, so it doesn't prune too hard to exclude any legit tuple from the final join result.

### False-Positive Rate $P_e$

In an array of $m$ bits,
a hash function sets a bit to $1$ with a probability of $P_{set} = \frac{1}{m}$.

For a Bloom filter with $k$ hash functions, the insertion of 

For a Bloom filter with $k$ hash functions with $n$ elements inserted, the possibility of seeing such a false positive is

$$P_e = [1 - (1 - \frac{1}{m})^{k\cdot n}]^{k}.$$

## Filter in Distributed Systems (2000s)
[Bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)

## Looking ahead with Filters (2010s)


## Accelarating Distributed Semi-join (2020-2025)

## What's next? (2026)


