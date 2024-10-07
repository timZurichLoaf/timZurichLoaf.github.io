---
layout: post
title: "Optimal Bushy Join Trees"
---

## Graph Representation of Join Query

Consider a query with only natural joins such as the SQL query below,


```sql
SELECT * FROM r1 JOIN  r2 JOIN r3 JOIN r4
```

$\Leftrightarrow \quad R(a, b, c, d, e) = R_1(a, b), R_2(b, c), R_3(c, d), R_4(d, e)$

By regarding each relation, such as $R_1(a, b)$, as a vertex and establishing an edge between vertices if they share some common variables, such a edge between $R_1(a, b), R_2(b, c)$, we find a graph representation of the query. The example above admits a chain of 4 vertices.

$$R_1 - R_2 - R_3 - R_4$$

In addition to the chain, we have also,
- Cycle Queries, such as $R = R_1(a, b), R_2(b, c), R_3(c, a)$
- Clique Queries, such as the 3-clique above
- Star Queries, such as $R = R_1(a, b), R_2(a, b), R_3(a, c)$

## Graph to Join Tree the Old Way

[Thomas Neumann's paper](https://dl.acm.org/doi/10.5555/1182635.1164207) evaluates two existing Dynamic Programming algorithms that look for the optimal join tree (query plan) by explioting the graph presentation with the focus on the # of (sub-)query plans enumerated.

The authors first introduce two key concepts (acronyms),
- **csg** for Connected SubGraph 
- **ccp** for Csg complete Pair

Considering the 4-relation chain query above, $R_1(a, b), R_2(b, c)$ is a **csg**, but $R_1(a, b), R_3(c, d)$ is not.

A **ccp** is a pair of **csg**s that cover all vertices without sharing any in common.

For example,
- $R_1(a, b), R_2(b, c)$ and $R_3(c, d), R_4(d, e)$ are a **ccp**
- But not $R_1(a, b), R_2(b, c), R_3(c, d)$ and $R_3(c, d), R_4(d, e)$ because of the common vertex $R_3$
- Not $R_1(a, b), R_2(b, c), R_4(d, e)$ and $R_3(c, d)$ either, because the former is not *connected*.

Now, the # of (sub-)query plans enumerated is exactly the # of **ccp**s enumerated.

The first algorithm $DPsize$ builds the following DP table in a bottom-up manner. Given the relations, it find the optimal 

{:class="table table-bordered"}
|               	|               	|               	|           	|           	|           	|
|-------------------	|---------------	|---------------	|-----------	|-----------	|-----------	|
| $R_1,R_2,R_3,R_4$ 	|               	|               	|           	|           	|           	|
| $R_1,R_2,R_3$     	| <span style="color:red">$R_2,R_3,R_4$</span> 	| <span style="color:grey">$R_3,R_4,R_1$</span> 	|           	|           	|           	|
| $R_1,R_2$         	| <span style="color:pink">$R_2,R_3$</span>     	| <span style="color:pink">$R_3,R_4$</span>     	| <span style="color:grey">$R_4,R_1$</span> 	| <span style="color:grey">$R_1,R_3$</span> 	| <span style="color:grey">$R_2,R_4$</span> 	|
| $R_1$             	| <span style="color:pink">$R_2$</span>         	| $R_3$         	| <span style="color:pink">$R_4$</span>     	|           	|           	|

The second algorithm $DPsub$ uses a bitvector of size $n$ to represent the queries contained in a subquery and builds the DP from left to right.

{:class="table table-bordered"}
| 0001  	| 0010  	| 0011       	| 0100  	| 0101           	| 0110       	| 0111            	| ... 	|
|-------	|-------	|------------	|-------	|----------------	|------------	|-----------------	|-----	|
| <span style="color:pink">$R_1$</span> 	| $R_2$ 	| <span style="color:pink">$R_1, R_2$</span> 	| <span style="color:pink">$R_3$</span> 	| <span style="color:grey">$R_1, R_3$</span> 	| <span style="color:pink">$R_2, R_3$</span> 	| <span style="color:red">$R_1, R_2, R_3$</span> 	| ... 	|

The authors conclude that $DPsub$ works better when the search space is dense, such as for clique queries, where most partitions of the graph tend to be connected, thus **csg**.

$DPsize$ works better when the searching space is sparse, such as for chain queries.

## Graph to Join Tree the New Way

The authors propose a universal solution $DPccp$ superior to $DPsize$ and $DPsub$ by enumerating only the valid **ccp**s without iterating over the duplicates or cartesian (cross) products.

$DPccp$ requires a breath-first numbering of all vertices (relations) before it explores the neighborhood of each vertex following a descending order their BFS indices. The neighborhood of a vertex is limited to itself and the vertices with larger BFS indices.

Let's bring up once more the chain query example in the beginning, $R(a, b, c, d, e) = R_1(a, b), R_2(b, c), R_3(c, d), R_4(d, e)$.

We rename the relations with breath-first numbering starting from the original $R_2(b, c)$ with the resulting query shown below.

$$R(a, b, c, d, e) = R'_2(a, b), R'_1(b, c), R'_3(c, d), R'_4(d, e)$$

For simplicity, we write $R_i'$ as $R_i$. The chain graph is now as follows.

$$R_2 - R_1 - R_3 - R_4$$

$DPccp$ first the neighborhood of the vertex with the largest BF number, namely $R_4$, while masking out all vertices with smaller BF numbers, namely $R_1, R_2$ and $R_3$.

$${\color{grey}R_2 - R_1 - R_3 - } {\color{red}R_4} \quad\quad \text{Output:}\, \{R_4\}$$

$DPccp$ moves on to explore $R_3$, while masking out $R_1, R_2$. In addition to the singleton set \{R_3\}, it also ouputs an $R_3$'s neighborhood $\{R_3, R_4\}$.

$${\color{grey}R_2 - R_1 -} {\color{red}R_3} - R_4 \quad\quad \text{Output:}\, \{R_3\}, \{R_3, R_4\}$$

When $DPccp$ explores $R_2$, while masking out $R_1$. Though visible at this moment, neither $R_3$ or $R_4$ is connected to $R_2$.  

$${\color{red}R_2} {\color{grey} - R_1 -} R_3 - R_4 \quad\quad \text{Output:}\, \{R_2\}$$

Finally, $DPccp$ reaches $R_1$ with everything in the sight range. The neighborhoods are explored in a depth-first fashion, resulting in an ascending order of their sizes.

$$R_2 - {\color{red}R_1} - R_3 - R_4 \quad\quad \text{Output:}\, \{R_1\}, \{R_1, R_2\}, \{R_1, R_3\}, \{R_1, R_3, R_4\}$$

We may quickly notice two things that $DPccp$ does very well.
- No cartesian (cross) products, such as $R'_2(a, b), R'_4(d, e)$, are enumerated. It checks only the **valid** **ccp**s. The more selective searching space sets it apart from $DPsize$ and $DPsub$;
- Each valid **ccp** is visited only once. No duplicates.


