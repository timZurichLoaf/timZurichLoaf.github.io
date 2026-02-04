---
layout: post
title: "[invisible] Optimization of Distributed Joins"
---

## Challenges of Joins
When we join the following three table $R(A, B) \bowtie S(B, C) \bowtie T(C)$, there is a decision to make about which two to join first.

<div style="text-align: center;">
<table style="display:inline-block; margin-right:40px; vertical-align:top; text-align:center;">
  <thead>
    <tr>
      <th colspan="2">R(A, B)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>1</td></tr>
    <tr><td>2</td><td>1</td></tr>
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
    <tr><td>1</td><td>3</td></tr>
    <tr><td>1</td><td>4</td></tr>
  </tbody>
</table>
<!--  -->
<table style="display:inline-block; vertical-align:top; text-align:center;">
  <tr><th>T(C)</th></tr>
  <tr><td>3</td></tr>
  <tr><td>5</td></tr>
</table>
</div>

$R(A, B) \bowtie S(B, C)$ creates in the big intermediate result below, where only two tuples highlighted 
in red eventually appear in the final join result.
It is even more undesirable in a distributed setting, because
of the steep network cost to transmit the redundant.

<div style="text-align: center;">
<table style="display:inline-block; margin-right:40px; vertical-align:top; text-align:center;">
  <thead>
    <tr>
      <th colspan="3">R(A, B) ⋈ S(B, C)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>1</td><td>1</td></tr>
    <tr><td>1</td><td>1</td><td>2</td></tr>
    <tr>
      <td><span style="color:red">1</span></td>
      <td><span style="color:red">1</span></td>
      <td><span style="color:red">3</span></td>
    </tr>
    <tr><td>1</td><td>1</td><td>4</td></tr>
    <tr><td>2</td><td>1</td><td>1</td></tr>
    <tr><td>2</td><td>1</td><td>2</td></tr>
    <tr>
      <td><span style="color:red">2</span></td>
      <td><span style="color:red">1</span></td>
      <td><span style="color:red">3</span></td>
    </tr>
    <tr><td>2</td><td>1</td><td>4</td></tr>
  </tbody>
</table>
<!-- </div> -->
<!--  -->
<!-- <div style="text-align: center;"> -->
<table style="display:inline-block; margin-right:40px; vertical-align:top; text-align:center;">
  <thead>
    <tr>
      <th colspan="3">S(B, C) ⋈ T(C)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><span style="color:red">1</span></td>
      <td><span style="color:red">3</span></td>
    </tr>
  </tbody>
</table>
</div>

An alternative choice to process $R(A, B) \bowtie S(B, C)$
first is arguably better, as it leaves the only tuple that
contributes to the final join result.




Assuming each table is stored on a different node,
we still need to ship many redundant tuples to facilitate that join.
Is there a way to avoid or at least cut down the redundant network cost?

## Exact Semi-joins

A semi-join $R \ltimes S$ keeps only the tuples in $R$ who also appear in $S$. Say we have two single-column tables $R(A)$ and $S(A)$.

<div style="text-align: center;">
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
</div>

<!-- --- -->

The semi-join $R \ltimes S$ keeps only $\color{red}{2, 4}$ in **R** who also appear in **S**.

[Yannakakis Algorithm](https://dl.acm.org/doi/10.5555/1286831.1286840) 
uses semi-joins to filter out **every** irrelevant tuple,
so that the join is executed later on the pruned tables with many fewer tuples.

Take the 3-table join $R(A, B) \bowtie S(B, C) \bowtie T(C)$ as an example again.

<div style="text-align: center;">
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
  <tr><td>5</td></tr>
</table>
</div>


The semi-join $S \ltimes T$ finds it enough to only consider the tuple $(1, 3)$ in $S(B, C)$.
The other semi-join $R \ltimes S$ finds $(1, 1)$ and $(2, 1)$ in $R(A, B)$.
After pruning, we only need to consider the highlighted tuples while executing the 3-table join.


Now the questions are
1. How to do a semi-join faster & more compactly?
2. In what order to perform those semi-joins?



## Compact Bloom Filter for Fast Membership Queries 

A semi-join is a batch of membership queries. 

[Bloom filter](https://dl.acm.org/doi/10.1145/362686.362692) is a space-efficient probabilistic data structure for fast membership queries.

Two key components of a Bloom filter are bit array and hash function.
Say we start with a bit array of size $10$ and $2$ hash functions,

$$h_1(x) = x \bmod 10;\quad h_2(x) = (x + 4) \bmod 10.$$

Initially, all the bits are set to $0$, meaning no element has been inserted yet.

{:class="table table-bordered table-hover table-sm"}
| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

To insert `1`,
because $h_1(1) = 1$ and $h_2(1) = 5$, 
the bits at positions **1** and **5** are set to $1$.

{:class="table table-bordered table-hover table-sm"}
| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | <span style="color:red">1</span> | 0 | 0 | 0 | <span style="color:red">1</span> | 0 | 0 | 0 | 0 |

To insert `3`,
because $h_1(3) = 3$ and $h_2(3) = 7$, 
the bits at positions **3** and **7** are set to $1$.

{:class="table table-bordered table-hover table-sm"}
| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 0 |

The query of `3` returns true positive,
as the bits at positions 3 and 7 are **both** $1$.


The query of `5` returns true negative,
as the bit at position $h_1(5) = 5$ is $1$ but that at $h_2(5) = 9$ is $0$.

<!-- {:class="table table-bordered table-hover table-sm"}
| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 1 | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 1 | 0 | <span style="color:blue">0</span> | -->

The query of `7` returns <span style="color:red">false positive</span>.
Though it is never inserted, 
the bits at positions $h_1(7) = 7$ and $h_2(7) = 1$ both have been set
to $1$ by the insertions of some other elements.

<!-- {:class="table table-bordered table-hover table-sm"}
| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | <span style="color:red">1</span> | 0 | 1 | 0 | 1 | 0 | <span style="color:red">1</span> | 0 | 0 | -->


The time & space efficiency of Bloom filter comes at the cost of accuracy. 
Hash collisions result in <span style="color:red">false positives</span>.

***Increasing the filter size reduces the false positive rate.***

In an array of $m$ bits,
a hash function supposedly maps uniformly to one of the $m$ bits.
So it sets a bit to $1$ with a probability of $P_{1} = \frac{1}{m}$
and leaves a bit unchanged with $P_{0} = 1 - \frac{1}{m}$.

Assuming $k$ independent hash functions of a Bloom filter,
the probability of some bit remains $0$ after inserting an element is

$$P_{0}^k = (1 - \frac{1}{m})^k.$$

After inserting all $n$ elements, the probability of some bit remains $0$ is

$$P_{0}^{k \cdot n} = (1 - \frac{1}{m})^{k \cdot n}.$$

Then the probability of a bit being set to $1$ is

$$P_{set} = 1 - P_{0}^{k \cdot n} = 1 - (1 - \frac{1}{m})^{k \cdot n}.$$

The query of an element that's never been inserted returns **false positives** if any of the $k$ bits it's hashed to has
been set to $1$. It gives us the **false-positive** rate, a.k.a.
the **error** rate,

$$P_e = P_{set}^k = [1 - (1 - \frac{1}{m})^{k\cdot n}]^{k}.$$

With the exponential approximation $\lim_{m \to \infty}(1 - \frac{1}{m})^m = e^{-m}$, 
we have

$$P_e \approx [1 - e^\frac{-k\cdot n}{m}]^{k}.$$

Given the number of elements $n$ and that of hash functions $k$,
increasing the filter size $m$ reduces the false positive rate $P_e$.

***Optimal Setup***

A Bloom filter is a bit array where each bit is binary, either 0 or 1.
It contains the maximum information per bit,
measured by [entropy](https://en.wikipedia.org/wiki/Entropy_in_thermodynamics_and_information_theory), when half-full, 
i.e. $P_{set} = \frac{1}{2}$.

$$1 - (1 - \frac{1}{m})^{k \cdot n} = \frac{1}{2}$$

$$\Downarrow$$

$${k \cdot n} \cdot \log(1 - \frac{1}{m}) = -\log(2)$$

Because $\lim_{m\to \infty}\log(1 - \frac{1}{m}) = - \frac{1}{m}$,

$$\Downarrow$$

$$m \approx \frac{{k \cdot n}}{\log(2)}.$$


## Distributed Approximate Semi-joins

The very purpose of semi-joins is to prune the tables.

If we don't need to filter out **every** irrelevant tuple, 
[Mackert and Lohman](https://dl.acm.org/doi/epdf/10.1145/16856.16863) finds the compact [Bloom filters](https://dl.acm.org/doi/10.1145/362686.362692) an answer to fast membership queries. Bloom filters are easy to compute locally on each node. Thanks to their compactness, they are cheap to transmit over the network in a distributed database. 

It only gives **false positives** for membership queries, 
so it doesn't prune too hard to exclude any legit tuple from the final join result.

For the 3-table join above, if we were using a Bloom filter built on $T(C)$ 
to do an **approximate semi-join** that filters out **some** irrelevant 
tuples from $S(B, C)$ but leaving a false-positive $\color{orange}(1, 4)$, 
the final join would remain correct.

<div style="text-align: center;">
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
    <tr><td><span style="color:orange">1</span></td><td><span style="color:orange">4</span></td></tr>
  </tbody>
</table>
<!--  -->
<table style="display:inline-block; vertical-align:top; text-align:center;">
  <tr><th>T(C)</th></tr>
  <tr><td><span style="color:red">3</span></td></tr>
  <tr><td>5</td></tr>
</table>
</div>

## Avoid Unnecessary Semi-joins


[Mullin](https://ieeexplore.ieee.org/document/52778)
notices that some semi-joins are totally unnecessary, like $S \ltimes R$
that does **NOT** filter out a single tuple from $S$.

His solution is to partition one big Bloom filter into multiple smaller ones,
send them one-by-one over the network and terminate this approximate semi-join between two nodes (supposedly hosting two tables) once
a small partitioned filter fails in filtering out **many enough** irrelevant tuples.

How many is enough? A rule of thumb is that 
the size of irrelevant tuples needs to outweigh that of the partitioned
filter to justify the communication cost.

How to partition? ***$k$ filters each of size $\frac{m}{k}$***.

The error rate mentioned above for a Bloom filter of size $m$ with $k$ hash functions 
and $n$ elements inserted is

$$P_e = [1 - (1 - \frac{1}{m})^{k\cdot n}]^{k}.$$

If we were using $k$ partitioned Bloom filters of smaller size $m' < m$ each with 
**only one** hash function, the error rate would be

$$P'_e = [1 - (1 - \frac{1}{m'})^{n}]^{k}.$$

To achieve the same error rate, let $P_{set} = P'_{set}$,

$$[1 - (1 - \frac{1}{m})^{k\cdot n}]^{k} = [1 - (1 - \frac{1}{m'})^{n}]^{k}$$

$$\Downarrow$$

$$(1 - \frac{1}{m'}) = (1 - \frac{1}{m})^{k}$$

By [Taylor Expansion](https://en.wikipedia.org/wiki/Taylor_series), 

$$\Downarrow$$

$$(1 - \frac{1}{m'}) = (1 - \frac{1}{m})^{k} = 1 - \frac{k}{m} + \frac{k(k-1)}{2m^2} + \cdots$$

The filter size is usually much larger than the number of hash functions $m \gg k$,

$$\Downarrow$$

$$(1 - \frac{1}{m'}) \approx 1 - \frac{k}{m} + O(\frac{1}{m^2})$$

$$\Downarrow$$

$$m' \approx \frac{m}{k}.$$

<!-- ## Adoption in Distributed Systems (2006)
[Bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) maintains a Bloom filter
for each logical storage unit, SSTables (Sorted String Tables),
to check whether it **might** contain any data for a specified row/column pair. 
This pratice turns out 
drastically reducing the number of disk seeks required for read operations.  -->

## Refining Approximate Semi-joins

Network is costly, while local computation is cheap. 
Upon receing a Bloom filter from the previous node, 
the current node can refine that filter locally with a cheap bitwise AND operations 
and pass on the refined to the next node.

Say we are joining three single-column tables, the first one with two tuples $1, 2$ and the second one with $2, 3$.

The two hash functions
$$h_1(x) = x \bmod 10;\quad h_2(x) = (x + 4) \bmod 10$$

would give us the Bloom filters below

{:class="table table-bordered table-hover table-sm"}
| BF1   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | <span style="color:red">1</span> | <span style="color:red">1</span> | 0 | 0 | <span style="color:red">1</span> | <span style="color:red">1</span> | 0 | 0 | 0 |

{:class="table table-bordered table-hover table-sm"}
| BF2   | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 0 | <span style="color:red">1</span> | <span style="color:red">1</span> | 0 | 0 | <span style="color:red">1</span> | <span style="color:red">1</span> | 0 | 0 |

Local bitwise AND gives us the refined filter as

{:class="table table-bordered table-hover table-sm"}
| BF1&2 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|------:|---|---|---|---|---|---|---|---|---|---|
| Bit   | 0 | 0 | <span style="color:red">1</span> | 0 | 0 | 0 | <span style="color:red">1</span> | 0 | 0 | 0 |

that contains the information about the tuples apearing in **both** tables
and can be passed it on to the third table.

[Ramesh](http://link.springer.com/10.1007/978-3-540-89737-8_15) sees two ways to refine the approximate semi-joins in 
a master-slaves (or user-sites) distributed database such as the one below, where $\text{Site}_k$ stores table $T_k$; $BF_{i\, j\, k}$ is
a Bloom filter refined by $T_i$, $T_j$, $T_k$; $RS_{i\, j\, k}$ is
the join result of $T_i$, $T_j$, $T_k$.

<img style='height: 85%; width: 85%; object-fit: contain' src="{{site.baseurl}}/assets/img/20260116_distributed_joins/ramesh_2008.png">

The scheme (a) refines the Bloom filters in one pass from $\text{Site}_1$, $\text{Site}_2$ to $\text{Site}_N$ in a cascading manner.
It then incrementally computes the join result in a reversed pass from $\text{Site}_N$ back to $\text{Site}_1$ before returning
the final join result to the user.

The scheme (b) refines the Bloom filters in two passes, first left-to-right from $\text{Site}_1$, $\text{Site}_2$ to $\text{Site}_N$ 
and then right-to-left. 
Each site then uses the highly refined filter at hand to return 
a pruned table to the user who eventually computes the final join result.

The network cost overweighs the local computation, therefore 
determining the order of those approximate semi-joins.
Because of the incremental computation of join result, 
the scheme (a) relies uses the order for semi-joins and the reversed 
order for joins. The scheme (b) has the flexibility of adopting a
different order when the user executes the final join.

## Predict Transfer for Distributed Joins

Joins are not always on the same attribute,
like the running example of 3-table join $R(A, B) \bowtie S(B, C) \bowtie T(C)$, where the local bitwise-AND operations
are not enough.

[Predicate Transfer](https://www.cidrdb.org/cidr2024/papers/p22-yang.pdf) extends the Bloom-filter-powered approximate semi-joins to different attributes across multiple tables.
For example, when we consider a left-to-right pass of semi-joins $S := S \ltimes R$, 
$T := T \ltimes S$ and assume no false positive. 
What happens behind the scene is a Bloom filter on $R(B)$
being shipped to $S(B, C)$ to filter out <del>$(3, 4)$</del> and
another Bloom filter on the filtered $S(C)$ being shipped to
$T(C)$ to filter out <del>$(5)$</del>.

<div style="text-align: center; white-space: nowrap;">

<table style="display:inline-block; vertical-align:top; text-align:center; margin-right:10px;">
  <thead>
    <tr>
      <th colspan="2">R(A, B)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>1</td></tr>
    <tr><td>2</td><td>1</td></tr>
    <tr><td>3</td><td>2</td></tr>
    <tr><td>4</td><td>5</td></tr>
  </tbody>
</table>
<span style="display:inline-block; vertical-align:middle; font-size:24px; margin:0 10px;">
  →
</span>
<table style="display:inline-block; vertical-align:top; text-align:center; margin-right:10px;">
  <thead>
    <tr>
      <th colspan="2">S(B, C)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>3</td></tr>
    <tr><td>1</td><td>2</td></tr>
    <tr><td>2</td><td>3</td></tr>
    <tr>
      <td style="text-decoration: line-through;">3</td>
      <td style="text-decoration: line-through;">4</td>
    </tr>
  </tbody>
</table>
<span style="display:inline-block; vertical-align:middle; font-size:24px; margin:0 10px;">
  →
</span>
<table style="display:inline-block; vertical-align:top; text-align:center;">
  <tr><th>T(C)</th></tr>
  <tr><td>3</td></tr>
  <tr><td style="text-decoration: line-through;">5</td></tr>
</table>
</div>

A right-to-left pass semi-joins $S := S \ltimes T$ and 
$R := R \ltimes S$ further filters out <del>$(1, 2)$</del> from $S(B, C)$
and <del>$(4, 5)$</del> from $R(A, B)$.

<div style="text-align: center; white-space: nowrap;">
<table style="display:inline-block; vertical-align:top; text-align:center; margin-right:10px;">
  <thead>
    <tr>
      <th colspan="2">R(A, B)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>1</td></tr>
    <tr><td>2</td><td>1</td></tr>
    <tr><td>3</td><td>2</td></tr>
    <tr>
      <td style="text-decoration: line-through;">4</td>
      <td style="text-decoration: line-through;">5</td>
    </tr>
  </tbody>
</table>
<span style="display:inline-block; vertical-align:middle; font-size:24px; margin:0 10px;">
  ←
</span>
<table style="display:inline-block; vertical-align:top; text-align:center; margin-right:10px;">
  <thead>
    <tr>
      <th colspan="2">S(B, C)</th>
    </tr>
  </thead>
  <tbody>
    <tr><td>1</td><td>3</td></tr>
    <tr>
      <td style="text-decoration: line-through;">1</td>
      <td style="text-decoration: line-through;">2</td>
    </tr>
    <tr><td>2</td><td>3</td></tr>
    <tr>
      <td style="text-decoration: line-through;">3</td>
      <td style="text-decoration: line-through;">4</td>
    </tr>
  </tbody>
</table>
<span style="display:inline-block; vertical-align:middle; font-size:24px; margin:0 10px;">
  ←
</span>
<table style="display:inline-block; vertical-align:top; text-align:center;">
  <tr><th>T(C)</th></tr>
  <tr><td>3</td></tr>
  <tr><td style="text-decoration: line-through;">5</td></tr>
</table>
</div>

Each table (node) receives some incoming filters on join attributes
to filter out its redundant tuples and generates some outgoing
filters on potentially different join attributes.

<div style="text-align: center;">
  <!-- <table style="display:inline-block; vertical-align:top; text-align:center; border-collapse:collapse;"> -->
  <table style="display:inline-block; margin: 0 20px; vertical-align:top; text-align:center; border-collapse:collapse;">
    <thead>
    <tr>
        <th></th>
        <th colspan="2">node 1</th>
        <th></th>
      </tr>
      <tr>
        <th colspan="2">$R_1(A, B)$</th>
        <!-- <th style="border-left:4px solid black;"></th> -->
        <th style="border-left:4px solid black;" colspan="2">$S_1(B, C)$</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>1</td><td>1</td>
        <!-- <td style="border-left:4px solid black;"></td> -->
        <td style="border-left:4px solid black;">1</td><td>3</td>
      </tr>
      <tr>
        <td>2</td><td>1</td>
        <!-- <td style="border-left:4px solid black;"></td> -->
        <td style="border-left:4px solid black;">1</td><td>2</td>
      </tr>
      <tr>
      <td colspan="2">↓</td><td colspan="2">↓</td>
      </tr>
      <tr>
      <td colspan="2">$BF_{R1}$</td><td colspan="2">$BF_{S1}$</td>
      </tr>
    </tbody>
  </table>
<!--  -->
</table>
  <!-- <table style="display:inline-block; vertical-align:top; text-align:center; border-collapse:collapse;"> -->
  <table style="display:inline-block; margin: 0 20px; vertical-align:top; text-align:center; border-collapse:collapse;">
    <thead>
    <tr>
        <th></th>
        <th colspan="2">node 2</th>
        <th></th>
      </tr>
      <tr>
        <th colspan="2">R(A, B)</th>
        <!-- <th style="border-left:4px solid black;"></th> -->
        <th style="border-left:4px solid black;" colspan="2">S(B, C)</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>3</td><td>2</td>
        <!-- <td style="border-left:4px solid black;"></td> -->
        <td style="border-left:4px solid black;">2</td><td>3</td>
      </tr>
      <tr>
        <td>4</td><td>2</td>
        <!-- <td style="border-left:4px solid black;"></td> -->
        <td style="border-left:4px solid black;">3</td><td>4</td>
      </tr>
    </tbody>
  </table>
</div>




## What's next? (2026)


