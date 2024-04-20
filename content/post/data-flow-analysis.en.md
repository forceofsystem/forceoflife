---
title: 'Data Flow Analysis'
date: 2024-03-05T23:16:20+08:00
draft: false
comments: true
toc: true
tags:
  - 编译原理
  - 优化
  - Compiler
  - Optimization
---

这篇文章是我在看 _Engineering A Compiler_ (EAC，橡书) 中第 9 章数据流分析的笔记，里面标有 Confused 的地方是我困惑的内容，包括过程间常量传播 (Jump Function)、加速DOM计算，之后如果搞懂了会写文章来补充。

<!--more-->

## Introdution

Requirements for optimization:

- find the points which can be improved
- the transformation is safe

## Iterative Data-Flow Analysis

For a forward data-flow problem, the algorithm should use an RPO computed on the CFG. By contrast, the algorithm should use an RPO computed on the reverse CFG to solve a backward data-flow problem.

> Reverse CFG: the CFG with its edges reversed. Normally, the compiler need to add a unique exit node so that the reverse CFG has a unique entry node.
> 

> Reverse Postorder (RPO): Label the nodes in the graph with their visitation order in a reverse postorder traversal which visits as many of a node’s predecessors as possible before visiting the node itself. **A node’s RPO number is simple $|N| + 1$ minus its postorder number, where *N* is the set of nodes in the graph.**
> 

### Dominance

Definition: node a dominates node b, if and only if a lies on every path from the entry of CFG to b.

> The dominance calculation is a forward data-flow problem, and relies only on the structure of the graph.
> 

$D_{OM} (b_i)$: a set which contains the names of all nodes that dominate $b_i$.

**Using the following equation to compute these sets:**

$D_{OM} (n) = \{n\} \cup \left( \mathop{\bigcap}\limits_{m\in preds(n)} D_{OM} (m) \right)$

with the initial conditions that $D_{OM} (n_0) = \{n_0\}$, and $\forall n ≠ n_0$, $D_{OM} (n) = N$, where N is the set of all nodes in the CFG.

**Three-step to solve the equation:**

1. build a CFG
2. gather initial information for each block
3. solve the equations to produce the $D_{OM}$ sets for each block

![Untitled](/2024/data-flow-analysis/images/Iterative-Dominance-Framework.png)

### Live-Variable Analysis

Type: a backward global data-flow problem

> The result of this analysis can be used for register allocation and construction of some variants of SSA form.
> 

**Equation:**

$L_{IVE}O_{UT}(n) = \mathop{\bigcup}\limits_{m \in succ(n)}(UEV_{AR}(m) \cup (L_{IVE}O_{UT}(m) \cap \overline{V_{AR}K_{ILL}(m)}))$

and the initial condition that $L_{IVE}O_{UT}(n) = \emptyset$, $\forall n$.

Three-step to compute the set:

1. Perform control-flow analysis to build a CFG.
2. Compute the values of the initial sets.
3. Apply the iterative algorithm.

**Using an RPO computed on the reversed CFG to solve this problem.**

### Limitations

1. The algorithm which computes the sets $L_{IVE}O_{UT}$, it implicitly assumes that execution can reach all of those successors. However, one or more of them may not be reachable.
2. Arrays and pointers, which force the analyzer to assume that every variable has been modified. We can solve this problem by guarateening the type safety or analysing the pointer variables.
3. Procedure calls. We also have to assume that the callee modifies all the variable which is accessible to both caller and callee and the call-by-reference parameters.

### Available Expressions

Definition: An expression e is available at point p in a procedure if and only if on every path from the procedure’s entry to p, e is evaluated and none of its constituent subexpressions is redefined between that evaluation and p.

Type: a forward data-flow problem.

The equation to compute the $A_{VAIL}I_N(n)$:

$A_{VAIL}I_N(n) = \mathop{\bigcap}\limits_{m \in preds(n)}(DEE_{XPR}(m) \cup (A_{VAIL}I_{N}(m) \cap \overline{E_{XPR}K_{ILL}(m)}))$

, the initial condition that $A_{VAIL}I_N(n_0) = \emptyset$, $A_{VAIL}I_N(n) = \{\  all \  expressions \ \}, \forall n ≠ n0$.

- $DEE_{XPR}(n)$: a set of downward exposed expressions in n. An expression $e \in DEE_{XPR}(n)$ if and only if block n evaluates $e$ and none of $e$’s operands is defined between the last evaluation of $e$ in $n$ and the end of $n$.
- $E_{XPR}K_{ILL}(n)$: this set contains all those expressions that are “killed” by  a definition in $n$. An expression is killed if and only if one or more of its operands are redefined in the block.

> An expression $e$ is available on entry to n if and only if it is available on exit from **each** of $n$’s predecessors in the CFG.
> 

> The $A_{VAIL}I_N(n)$ can be used to perform *global common subexpression elimination* and lazy code motion.
> 

### Reaching Defnitions

Purpose: to find the set of definitions that reach a block.

Domain of $R_{EACHES}$ is the set of definitions in procedure.

A definition d of some variable v reaches operation i if and only if i reads the value of v and there exists a path form d to i that doesn’t define v.

The compiler annotates each node n in the CFG with a set $R_{EACHES}(n)$, computed as a forward data-flow problem:

$R_{EACHES}(n) = \emptyset$, $\forall n$

$R_{EACHES}(n) = \mathop{\bigcup}\limits_{m \in preds(n)} ( DED_{EF}(m) \cup (R_{EACHES}(m) \cap \overline{D_{EF}K_{ILL}(m)}))$

$DED_{EF}(m)$ is the set of downward-exposed definitions in m: those definitions in m for which the defined name is not subsequently redefined in m.

$D_{EF}K_{ILL}(m)$ contains all the definition points that are obscured by a definition of the same name in m. Thus $\overline{D_{EF}K_{ILL}(m)}$ consists of the definition points that are not obsucred in m.

> Computing $DED_{EF}$ and $D_{EF}K_{ILL}$ requires a mapping from names to definiton points. It’s more complex to gather the initial information than it is for live variables.
> 

### Anticipable Expressions

Definition: 

An expression $e$ is considered *anticipable* on exit from block b if and only if

1. every path that leaves b evaluates and subsequently use $e$, and
2. evaluating $e$ at the end of $b$ would produce the same result as the first evaluation of $e$ along each of those paths. (this property corresponds to the term “anticipable”)

Domainof the problem: the set of expressions.

**Equation:**

$A_{NT}O_{UT}(n_f) = \emptyset$

$A_{NT}O_{UT}(n) = \{\ all \  expressions \ \}, \forall n ≠ n_f$

$A_{NT}O_{UT}(n) = \mathop{\bigcap}\limits_{m \in succ(n)}(UEE_{XPR}(m) \cup (A_{NT}O_{UT}(m) \cap \overline{E_{XPR}K_{ILL}(m)}))$

$UEE_{XPR}(m)$ is the set of upward-exposed expressions—those used in $m$ before they are killed.

> The results of anticipability analysis are used in code motion both in lazy code motion and code hoisting.
> 

### Interprocedural Summary Problems (Confused)

Problem: In the absence of specific information about the call, the compiler must make worst-case assumptions that account for all the possible actions of the callee, or any procedures that it, in turn, calls.

To limit such impact, the classic summary problems compute the set of variables that might be modified as a result of the call and that might be used as a result of the call.

The *intreprocedural may modify problem* annotates each call site with a set of names that the callee, and procedures it calls, might modify.

> The result of this analysis can be used for global constant propagation.
> 

**Equation:**

$M_{AY}M_{OD}(p) = L_{OCAL}M_{OD}(p) \cup \left( \mathop{\bigcup}\limits_{e = (p,q)} ubind_e(M_{AY}M_{OD}(q))\right)$

For a call-graph edge $e = (p,q)$, $unbind_e(x)$ maps each name in x from the name space of q to the name space of p, using the bindings at the specific call site that coreesponds to $e$.

$L_{OCAL}M_{OD}(p)$ contains all the names modified locally in p that are visible outside p.

- It’s computed as the set of names defined in p minus any names that are stricly local to p.

The compiler can also compute information on what variables might be referenced as a result of executing a procedural call, which is called the *interprocedural may reference problem*.

## Static Single-Assignment Form

Purpose: to limit the number of analyses.

- Solution: building a variant form that encodes both data flow and control flow directly in the IR.

$\phi$-function: this function combines the values from distinct edges. When control enters a block, all the $\phi$-functions in the block execute, **concurrently**. They evaluate to the argument that corresponds to the edge along which control entered the block.

Two rules hold after the transformation to SSA:

1. each definition in the procedure has a unique name, and 
2. each use refers to a single definition.

Steps to transform a procedure into SSA form:

1. Insert the appropriate $\phi$-functions for each variable into the code;
2. Rename variables with subscripts to make the two rules hold.

**Maximal SSA form**: a naive algorithm to convert IR to SSA, which creates many extraneous $\phi$-funtions.

### Dominator Trees

Strictly dominate set: $D_{OM}(n)-n$

Immediate dominator: the node in strictly dominate set that is closest to n, denoted $ID_{OM}(n)$.

> The entry node has no immediate dominator.
> 

Dominator tree: it contains all node of a flow graph, and there is an edge from m to n if m is $ID_{OM}(n)$.

The dominator tree contains both the $ID_{OM}$ information and the $D_{OM}$ sets for each node. Each node lies on the path from the root of dominator tree to n, inclusive of both the root and n.

### Dominance Frontiers

A definition in node n forces a corresponding $\phi$-function at any join point m where

1. n dominates a predecessor of m ($q \in preds(m)$ and $n \in D_{OM}(q)$), and
2. n doesn’t strictly dominate m.

Dominance frontier: the collection of nodes *m* that have above property with respect to n, denoted $DF(n)$.

The algorithm to compute dominance frontiers is based on three observations as following:

1. Node in a  DF set must be join points in the graph.
2. For a join point j, each predecessor k of j must have $j \in DF(k)$.
3. If $j \in DF(k)$ for some predecessor k, then j must also be in DF(l) for each $l \in D_{OM}(k)$, unless $l \in D_{OM}(j)$.

![DF set](/static/2024/data-flow-analysis/images/DF.png)

### Placing $\phi$-Functions

To improve the naive algorithm, the basic idea is that a definition of x in block b forces a $\phi$-function at every node in DF(b). But we can continue to improve it, because a variable that is only **live** within a single block can never have a live $\phi$-function.

Globals set: a set of names which are live across multiple blocks.

- In each block, it looks for names with upward-exposed uses — the $UEV_{AR}$ set from the live-variables calculation.
- With the globals set, the compiler can insert $\phi$-functions for those names within it and ignore any name that is not in it.

Blocks set: a list of blocks for each name that contain a definition of that name.

- These block lists serve as an initial worklist for the $\phi$-insertion algorithm.

Algorithm to find the sets:

![Globals and blocks](/2024/data-flow-analysis/images/Globals-Blocks.png)

and to rewrite the code:

![Untitled](/2024/data-flow-analysis/images/Insertion.png)

> Since all the $\phi$-functions in a block execute concurrently, the order of their insertion is insignificant.
> 

> However, the distinction between local names and global names is not sufficient to avoid all dead $\phi$-functions. To further eliminate the extraneous insertions, the compiler can construct $L_{IVE}O_{UT}$ sets and add a test based on liveness to the inner loop of the insertion algorithm. The SSA form generated by that modification is called pruned SSA form.
> 

There are two ways to improve the efficiency:

1. The algorithm should avoid placing any block on the worklist more than once per global name, which can be solved by keeping a checklist of blocks that have already been processed. For implementation, it can use a sparse set to save the space.
2. The compiler can maintain a checklist of blocks (a single sparse set, which reinitialized along with WorkList) that already contain $\phi$-functions for a given blocks.

### Renaming

![Renaming](/2024/data-flow-analysis/images/Renaming.png)

This algorithm renames both definitions and uses in a preorder walk over the procedure’s dominator tree. It rewrites the operands with current SSA names, then it creates a new SSA name for the result of the operation. After rewriting the operands and definition, the algorithm rewrites the appropriate $\phi$-function parameters in each CFG successor of the block, using the current SSA names.

Finally, it recurs on any children of the block in the dominator tree.

Moreover, the algorithm maintains a stack which holds the subscript of the name’s current SSA name.

It also maintain a counter to ensure that each definition receives a unique SSA name.

**The following things makes me confused.**

> The compiler must assign an ordinal parameter slot in those $\phi$-functions for b. When we draw the SSA form, we always assume a left-to-right order that matches the left-to-right order in which the edges are drawn. 
Internally, the compiler can number the edges and parameter slots in any consistent fashion that produces the desired result. This requires cooperation between the code that builds the ssa form and the code that builds the cfg. (For example, if the CFG implementation uses a list of edges leaving each block, the order of that list can determine the mapping.)
> 

We can use a method to limit the max size of stack to the depth of the dominator tree.

> *NewName* may overwrite the same stack slot multiple times within a single block. But it requires another mechanism for determining which stacks to pop on exit from a block.
> 
> 
> *NewName* can thread together the stack entries for a block. Rename can use the thread to pop the appropriate stacks.
> 

### Translation Out of SSA Form

Reason: modern processors do not implement $\phi$-functions. The compiler would produce incorrect code when it simply drops the subscripts, reverts to base names, and deletes the $\phi$-functions if the code has been rearranged.

The compiler can keep the SSA name space instact and replace each $\phi$-function with a set of copy operations—one along each incoming edge.

> Critical Edge: In a CFG, an edge whose source has multiple successors and whose sink has multiple predecessors is called acritical edge.
> 

If a block has multiple successors, the compiler cann’t simply insert copies directly into it. To remedy this problem, the compiler can split the edge, insert a new block between the nodes, and place the copies in that new block.

**The Lost-Copy Problem**

The lost-copy problem arises from the combination of copy folding and critical edges that cann’t be split.

For example:

![out-of-ssa-wrong](/2024/data-flow-analysis/images/out-of-ssa-wrong.png)

Panel a assigns z the second to last value of i; the code in panel c assigns $z_0$ the last value of i. With the critical edge split, as in panel d, copy insertion produces the correct behavior. However, it adds a jump to every iteration of the loop.

![split](/2024/data-flow-analysis/images/split-critical.png)

The compiler can avoid this problem by checking the liveness of the target name for each copy that it tries to insert during out-of-SSA translation. When it discovers a copy target(in this example is $i_1$) that is live, it must preserve the living value in a tempory name and rewrite subsequent uses to refer to the tempoary name.

> This step can be done with an algorithm modelled on the renaming step.
> 

![correct](/2024/data-flow-analysis/images/correct-insertion.png)

**The swap problem**

Reason: all $\phi$-functions in a block execute concurrently, but the assignments execute serially.

For example:

![naive-problem](/2024/data-flow-analysis/images/naive-insertion.png)

The straightford fix for this problem is to adopt a two-stage copy protocol.

- The first stage copies each of the $\phi$-function arguments to its own temporary name, simulating the behavior of the original $\phi$-functions.
- The second stage then copies those values to the $\phi$-function targets.

> For the example, it would require four assignments: $s = y_1$, $t = x_1$, $x_1 = s$, $y_1 = t$.
> 

Unfortunately, this solution doubles the number of copy operations required to translate out of SSA form.

In fact, this problem can arise without a cycle of copies. All it takes is a set of $\phi$-functions that have, as inputs, variables defined as outputs of other $\phi$-functions in the same block. Though it can be fixed easily by carefully arranging the order of inserted copies.

### Using SSA Form (Simply Sparse Constant Propagation Algorithm)

semilattice: A semilattice consists of a set $L$ of values and a meet operator, $\wedge$. The meet operator must be idempotent, commutative and associative; it imposes an order on the elements of $L$ as follows:

$a ≥ b$ if and only if $a \wedge b = b$ and

$a > b$ if and only if $a > b$ and $a ≠b$

- bottom element: $\bot$, with the properties that $\forall a \in L, a \wedge \bot = \bot, and \ \forall \ a \in L, a ≥ L$
- top element: $\top$, with the properties that $\forall a \in L, a \wedge \top = a, and \ \forall \ a \in L, T ≥ a$

The semilattice for a single SSA name:

![Semilattice](/2024/data-flow-analysis/images/Semilattice.png)

**Simply Sparse Constant Propagation Algorithm:**

![SSCP](/2024/data-flow-analysis/images/SSCP.png)

Initialization phase:

- If n is defined by a $\phi$-function, SSCP sets *Value(n)* to $\top$. (Optimistic)
- If n is a known constant $c_i$, SSCP sets *Value(n)* to $c_i$.
- If n’s value cannot be known, SSCP sets *Value(n)* to $\bot$.

Propagation phase:

- For a $\phi$-function, the result is simply the meet of the lattice values of all the $\phi$-function’s arguments.
- For other kinds of operations, the compiler must apply operator-specific knowledge.

**Because a SSA name can have one of three initial values, so each of them appears on the worklist at most twice.**

Optimistic algorithm: algorithm that begin with the value $\top$ rather than $\bot$.

> In opposition to above, algorithm that begin with the value $\bot$ is ‘pessimistic’.
> 

## Interprocedural Analysis

The inefficiencies introduced by procedure calls appear in two distinct form:

- loss of knowledge in single-procedure analysis and optimization that arise from the presence of a call site in the region being anaylzed and transformed, and
- specific overhead introduced to maintain the abstractions inherent in the procedure call.

### Call-Graph Construction

Normally, the call-graph construction’s limiting factor is just the cost of scanning procedures to find the call sites. However, some programming language features may make the construction harder.

- procedure-valued variables: the compiler must analyze the code to estimate the set of potential callees at each call site that invokes a procedure-valued variable. If it just use a simple analog of global constant propagation, it will build extraneous edge. To build the precise call graph, it must track sets of parameters that are passed together, along the same path. The algorithm could then consider each set independently to derive the precise graph.
- contextually-resolved names:
    - inheritance hierarchy in oriented-object;
    - a language that allows the program to import either executable code or new class at runtime.
        - To solve this problem, the compiler must construct a conservative call graph that reflects the complete set of potential callees at each call site.
- control-flow graph that has multiple entry and/or exit.
    - $M_{AY}M_{OD}$ analysis.

### Interprocedural Constant Propagation

Goal: Discovering situations where a procedure always receives a known constant value or where a procedure always returns a known constant value.

Three problems of interprocedural constant propagation:

- discovering an initial set of constants;
    - simple method: Recognize literal constant values used as parameters.
    - more effect and expensive way: Use a full-fledged global constant propation step.
- propagating known constant values around the call graph;
    - This portion of the analysis resembles the iterative data-flow algorithms.
- modelling transmission of values through procedures.
    - The compiler builds a small model for each actual parameter, call *jump function*. Each jump function, ${J_s}^x$relies on the values of some subset of the formal parameters to the procedure p that contains s, denoted $Support({J_s}^x)$.

Algorithm:

![IPO-CP](/2024/data-flow-analysis/images/IPO-CP.png)

> Similar to the above propagation algoritm, the worklist should be a sparse set or something like it.
> 

**Jump Function Implementation (Confused)**

The implementation of jump functions is abundant, but all of them must hold following principles:

1. If the analyzer determines that parameter x at call site s is a known constant c, then ${J_s}^x = c$, and $Support({J_s}^x) = \emptyset$.
2. If $y \in Support({J_s}^x)$ and $Value(y) = \top$, then ${J_s}^x = \top$.
3. If the analyzer determines that the value of ${J_s}^x$ cannot be determinded, then ${J_s}^x = \bot$.

**Two aspects to extend the algorithm**

1. We can also construct *return jump functions* to model the values returned from callee to caller. The algorithm can treat return jump functions in the same way that it handled ordinary jump functions. And the programmer need to avoid creating cycles of return jump functions that diverge (e.g. for a tail-recursive procedure).

> Return jump functions are particularly important for routines that initialize values such as setting initial values for an object or class in Java.
> 
1. To extend the algorithm to cover a larger class of variables, the compiler can simply extend the vector of jump functions.

## Advanced Topic

### Speeding up the Iterative Domiance Framework (Confused)

We can use the $ID_{OM}$ sets as a proxy for the $D_{OM}$ sets, provided we can provide efficient methods to initialize the sets and to intersect them. We can use a two-pointer algorithm to identify the common suffix.

![Advanced-Topic](/2024/data-flow-analysis/images/Advanced-Topic.png)

This improved algorithm efficient, it halts in two passes on any reducible graph, in more than two passes on any irreducible graph.
