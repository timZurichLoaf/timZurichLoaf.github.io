---
layout: post
title: "Optimal Bushy Join Trees"
---

## Graph Representation of Join Query

Consider a query with only natural joins such as the SQL query below,

<div align="center">
```SQL
SELECT * FROM r1 JOIN  r2 JOIN r3 JOIN r4
```
</div>

$$\Updownarrow$$


$$R(a, b, c, d, e) = R_1(a, b), R_2(b, c), R_3(c, d), R_4(d, e)$$

By regarding each relation, such as $$R_1(a, b)$$, as a vertex and establishing an edge between vertices if they share some common variables, such a edge between $R_1(a, b), R_2(b, c)$, we find a graph representation of the query. The example above admits a chain of 4 vertices.

$$R_1 - R_2 - R_3 - R_4$$

In addition to the chain, we have also,
- Cycle Queries, such as $R = R_1(a, b), R_2(b, c), R_3(c, a)$
- Clique Queries, such as the 3-clique above
- Star Queries, such as $R = R_1(a, b), R_2(a, b), R_3(a, c)$

## Graph to Join Tree the Old Way

[Thomas Neumann's paper](https://dl.acm.org/doi/10.5555/1182635.1164207) evaluates two existing Dynamic Programming algorithms that look for the optimal join tree (query plan) by explioting the graph presentation with the focus on the # of (sub-)query plans enumerated.

The authors first introduce two key concepts,
- **csg** for Connected SubGraph 
- **ccp** - Csg complete Pair

Considering the 4-relation chain query above, $R_1(a, b), R_2(b, c)$ is a **csg**, but $R_1(a, b), R_3(c, d)$ is not.

A **ccp** is a pair of **csg**s that cover all vertices without sharing any in common.

For example,
- $R_1(a, b), R_2(b, c)$ and $R_3(c, d), R_4(d, e)$ are a **ccp**
- But not $R_1(a, b), R_2(b, c), R_3(c, d)$ and $R_3(c, d), R_4(d, e)$ because of the common vertex $R_3$
- Not $R_1(a, b), R_2(b, c), R_4(d, e)$ and $R_3(c, d)$ either, because the former is not *connected*.

Now, the # of (sub-)query plans enumerated is exactly the # of **ccp**s enumerated.

The first algorithm $DPsize$ builds the following DP table in a bottom-up manner. Given the relations, it find the optimal 

|                   	|               	|               	|           	|           	|           	|
|-------------------	|---------------	|---------------	|-----------	|-----------	|-----------	|
| $R_1,R_2,R_3,R_4$ 	|               	|               	|           	|           	|           	|
| $R_1,R_2,R_3$     	| <span style="color:red">$R_2,R_3,R_4$</span> 	| ~~$R_3,R_4,R_1$~~ 	|           	|           	|           	|
| $R_1,R_2$         	| <span style="color:pink">$R_2,R_3$</span>     	| <span style="color:pink">$R_3,R_4$</span>     	| ~~$R_4,R_1$~~ 	| ~~$R_1,R_3$~~ 	| ~~$R_2,R_4$~~ 	|
| $R_1$             	| <span style="color:pink">$R_2$</span>         	| <span style="color:pink">$R_3$</span>         	| $R_4$     	|           	|           	|

The second algorithm $DPsub$ uses a bitvector of size $n$ to represent the queries contained in a subquery and builds the DP from left to right.

| 0001  	| 0010  	| 0011       	| 0100  	| 0101           	| 0110       	| 0111            	| ... 	|
|-------	|-------	|------------	|-------	|----------------	|------------	|-----------------	|-----	|
| <span style="color:pink">$R_1$</span> 	| $R_2$ 	| <span style="color:pink">$R_1, R_2$</span> 	| <span style="color:pink">$R_3$</span> 	| ~~$R_1, R_3$~~ 	| <span style="color:pink">$R_2, R_3$</span> 	| <span style="color:red">$R_1, R_2, R_3$</span> 	| ... 	|

The authors conclude that $DPsub$ works better when the search space is dense, such as for clique queries, where most partitions of the graph tend to be connected, thus **csg**.

$DPsize$ works better when the searching space is sparse, such as for chain queries.

## Graph to Join Tree the New Way

The authors propose a universal solution $DPccp$ superior to $DPsize$ and $DPsub$ by considering only the valid **ccp**s.

To be continued...




