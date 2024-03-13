---
title: 'Scalar Optimization'
date: 2024-03-14T02:43:34+08:00
draft: false
comments: true
toc: true
catagories:
  - 编译原理
  - 笔记
tags:
  - 数据流分析
  - 优化
---

This blog is a note of reading chapter 10 of EAC (Scalar Optimization), which contains not only the content in the book but my understanding. Ultimately, I have finished the reading of the whole book in two months, and will continue to read the dragon book and _Advanced Compiler Design and Implementation_. I will also write some notes about those books.

<!--more-->

## Overview

Scalar optimization: opimization of code along a single thread of control.

Goals of optimizer:

- Rewriting the code into a more efficient or more effective form.
- Producing highly efficient code on a much larger set of inputs.
- Robust

There are two machanisms to achieve these goals:

- It eliminates unnescessarys overhead introduced by programming language abstractions, and
- it matches the needs of the resulting program to the available hardware and software resources of the target machine.

> Machine independent: A transformation that improves code on most target machines is considered machine independent.
Machine dependent: A transformation that relies on knowledge of the target processor is considered machine dependent.
> 

**Most optimizers are built as a series of passes.** Each pass takes code in IR form as its input, and produces a rewritten version of the IR code as its output. This structure creates a natural way for the compiler to provide different levels of optimization; each level specifies a set of passes to run.

We concern, in general, five specific effects of the transformation:

- Eliminate useless and unreachable code
- Move code
- Specialize a computation
- Eliminate a redundant computation
- Enable other transformation

## Death Code Elimination

Useless code (death code): An operationis useless if no operation uses its result, or if all uses of the result are, themselves dead.

Unreachable code: An operation is unreachable if no valid control-flow path contains the operation.

> The source of these code: macro expansion or naive translation in the front end.
> 

> These code arises in most programs as the direct result of optimization in the compiler.
> 

**Branch, or conditionnal branch, is different to jump, or unconditional jump.**

### Eliminating Useless Code

Critical operation: an operation is ***critical*** if:

- it sets return values for the procedure;
- it’s an input/output statement, or
- it affects the value in a storage location that may be accessible from outside the current procedure (like pointers).

The algorithm for eliminating useless code performs two pass over the code, like the mark-sweep gc.

- first pass:
    - Clear all the mark fields and mark critical operations as “useful”
    - Trace the operands of useful operations back to their definitions and marks them as “useful”
- second pass: walk the code and remove any operation not marked.

**The algorithm assumes that the code is in SSA form.**

![two-stage](/2024/scalar-optimization/two-stage.png)

For control-flow operations, the treatment of the algorithm is more complex than normal operations.

> Postdominance: In a CFG, *j* postdominates *i* if and only if every path from i to the exit node passes through j.
Reverse dominance frontier: dominance frontiers computed on the reverse CFG, denoted $RDF(j)$.

Control-dependence: in a CFG, node j is control-dependent on node i if and only if

1. There exists a nonnull path from i to j such that j postdominates every node on the path **after i**.
2. j **doesn’t strictly** postdominate i.

> In other words, two or more edges leaves block i. One or more edges leads to j and one or more edges do not.
Actually, I think this explanation is easier to understand than the former, but, it’s not a formal definition, hhh.
> 

When *Mark* marks an operation in block b as useful, it visits every block in b’s $RDF$ and marks their block-ending branches as useful. (the last part of *Mark* routine)

*Sweep* replaces any unmarked branch with a jump to its first postdominator that contains a marked operation. If the branch is unmarked, then its successors, down to its immediate postdominator, contain no useful operations. (Otherwise, when those operations were marked, the branch would have been marked.)

> To find the nearest useful postdominator, the algorithm can walk up the postdominator tree until it finds a block that con- tains a useful operation.
> 

### Eliminating Useless Control Flow

If the compiler includes optimizations that can produce useless control flow as a side effect, then it should include a pass that simplifies the cfg by eliminating useless control flow.

The *clean* algorithm uses four transformations, which are applyed in the following order:

1. Fold a Redundant Branch

![fold-redundant](/2024/scalar-optimization/fold-redundant-branch.png)

2. Remove an Empty Block

![remove-empty-block](/2024/scalar-optimization/remove-empty-block.png)

3. Combine Blocks

![combine-blocks](/2024/scalar-optimization/combine-blocks.png)

4. Hoist a Branch: If *clean* finds a block $B_i$ that ends with a jump to an empty block $B_j$ and $B_j$ ends with a branch. Clean can replace the block-ending jump in $B_i$ with a copy of the branch from $B_j$.

- $B_i$ cann’t be empty, and
- $B_i$ cann’t be $B_j$’s sole prodecessor.

![hoist-branch](/2024/scalar-optimization/hoist-branch.png)

A systematic C*lean* implementation traverses the graph in postorder, so that $B_i$’s successors are simplified before $B_i$ unless the successor lies along a back edge with respect to the postorder number- ing. This method reduces the number of times that the implementation must move some edges.

![DCE](/2024/scalar-optimization/dce.png)

If the CFG contains back edges, then a pass of *Clean* may create unprocessed successors along the back edges. For this reason, *Clean* repeats the transformation sequence iteratively until the CFG stops changing.

With the *Dead* routine’s help, *Clean* can elinimate an empty loop which can’t be done by itself instead of using a specific transformation to handle it.

### Eliminating Unreachable Code

A block can be unreachable for two distinct reasons:

1. there may be no path through the CFG that leads to the block, or 
2. the paths that reach the block may not be executable.

The former case is easy to handle with the same “mark-sweep” algorithm. However, the latter case requires the compiler to reason about the values of expressions that control branches.

> If the source language allows arithmetic on code pointers or labels (such as C’s pointers), the compiler must preserve all blocks. Otherwise, it can limit the preserved set to blocks whose labels are referenced.
> 

## Code Motion

### Lazy Code Motion

LCM performs loop-invariant code motion. It uses data-flow analysis to discover both operations that are candidates for code motion and locations where it can place those operations.

**The algorithm operates on the IR form of the program and its CFG, rather than on SSA form.**

> Redundant: an expression $e$ is redundant at p if it has already been evaluated on every path that leads to p.
Partially redundant: an expression $e$ is partially redundant at p if it occurrs on some, but not all, paths that reach p.
> 

LCM combines code motion with elimination of both redundant and partially redundant computations. For the partially redundant expressions, LCM inserts an edge on the path where p doesn’t occur to make the computation fully redundant.

#### **Code Shape**

LCM moves expression evaluations, not assignments.

> Program variables are set by register-to- register copy operations. Variables have lower subscripts than any expression, and that in any operation other than a copy, the defined register’s subscript must be larger than the subscripts of the operation’s arguments.
> 

#### **Available Expressions**

LCM needs availability at the end of the block, so it computes $A_{VAIL}O_{UT}$ rather than $A_{VAIL}I_N$.

> An expression $e$ is available on exit from block $b$ if, along every path from $n_0$ to $b$, $e$ has been evaluated and none of its arguments has been subsequently defined.
> 

LCM computes $A_{VAIL}O_{UT}$ as follows:

$A_{VAIL}O_{UT}(n_0) = \empty$

$A_{VAIL}O_{UT}(n) = \{\ all\ expressions\ \}$, $\forall n ≠ n_0$

and then iteratively evaluates the following equation until it reaches a fixed point:

$A_{VAIL}O_{UT}(n) = \mathop{\bigcap}\limits_{m \in preds(n)}(DEE_{XPR}(m) \cup (A_{VAIL}O_{UT}(m) \cap \overline{E_{XPR}K_{ILL}(m)}))$

If an expression $e \in A_{VAIL}O_{UT}(b)$, the compiler could place an evaluation $e$ at the end of block $b$ and obtain the result produced by its most recent evaluation on any control-flow path from $n_0$ to b.

**In this light, $A_{VAIL}O_{UT}$ sets tell the compiler how far forward in the CFG it can move the evaluation of $e$, ignoring any uses of $e$.**

#### **Anticipable Expressions**

Definition: an expression is anticipable at point $p$ if and only if it is computed on every path that leaves p and produces the same value at each of these compuations.

The initial condition of the sets is:

$A_{NT}O_{UT}(n_f) = \empty$

$A_{NT}O_{UT}(n) = \{\ all\ expressions\ \}$, $\forall n ≠ n_f$

Next, it solve the follow fixed-point problems:

$A_{NT}I_{N}(m) = \mathop{\bigcap}\limits_{m \in succ(n)}(UEE_{XPR}(m) \cup (A_{NT}O_{UT}(m) \cap \overline{E_{XPR}K_{ILL}(m)}))$

$A_{NT}O_{UT}(n) = \mathop{\bigcap}\limits_{m \in succ(n)} A_{NT}I_N(m)$, $n ≠ n_f$

$A_{NT}O_{UT}$ provides information about the safety of hoisting an evaluation to either the start or the end of the current block. If $x \in A_{NT}O_{UT}(b)$, then the compiler can place an evaluation of $x$ at the end of b, with two guarantees:

1. The evaluation at the end of b will produce the same value as the next evaluation of $x$ along any execution path in the procedure.
2. Along any execution path leading out of $b$, the program will evaluate $x$ before redefining any of its arguments.

#### **Earliest Placement**

> To simplify the placement equations, LCM assumes that it will place the evaluation on a CFG edge rather than at the start or end of a specific block.
> 

Computing an edge placement lets the compiler defer the decision to place the evaluation **at the end of the edge’s source**, **at the start of its sink**, or **in a new block in the middle of the edge**.

$E_{ARLIEST}$ set: for a CFG edge <i, j>, an expression $e$ is in $E_{ARLIEST}(i, j)$ if and only if the compiler can legally move $e$ to <i, j>, and cannot move it to any earlier edge in the CFG.

The $E_{ARLIEST}$ equation encodes this condition as the intersection of three terms:

$E_{ARLIEST}(i, j) = A_{NT}I_N(j) \cap \overline{A_{VAIL}O_{UT}(i)} \cap (E_{XPR}K_{ILL}(i) \cup \overline{A_{NT}O_{UT}(i)})$

These terms define an earliest placement for $e$ as follows:

1. $e \in A_{NT}I_{N}(j)$ means that the compiler can safely move $e$ to the head of $j$. 
2. $e \notin A_{VAIL}O_{UT}(i)$ shows that no prior computation of $e$ is available on exit from $i$.
3. If $e \in E_{XPR}K_{ILL}(i)$, the compiler cann’t move $e$ through block $i$ because of a definition in $i$. If $e \notin A_{NT}O_{UT}(i)$, the compiler cann’t move $e$ into $i$ because $e \notin A_{NT}I_{N}(k)$ for some edge <i, k>. If either is true, then $e$ can move no further than <i, j>

#### Later Placement

This problem determines when an earliest placement can be defered to a later point in the CFG while achieving the same effect. It’s formulated as a forward data-flow problem on the CFG with a set $L_{ATER}I_N(n)$ associated with each node and another set $L_{ATER}(i, j)$ associated with each edge <i, j>.

LCM initializes the $L_{ATER}I_N$ sets as follows:

$L_{ATER}I_N(n_0) = \empty$

$L_{ATER}I_N(n) = \{\ all\ expressions\ \}$, $\forall n ≠ n_0$

Next, it iteratively computes $L_{ATER}I_N$ and $L_{ATER}$ sets for each block.

$L_{ATER}I_N(j) = \mathop{\bigcap}\limits_{m \in preds(j)} L_{ATER}(i, j), j \not= n_0$

$L_{ATER}(i, j) = E_{ARLIEST}(i, j) \cup (L_{ATER}I_N(i) \cap \overline{UEE_{XPR}(i)})$

An expression $e \in L_{ATER}I_N(k)$ if and only if:

- every path that reaches k includes an edge <p, q> such that $e]in E_{ARLIEST}(p, q)$, and
- the path from q to k
    - neither redefines $e’s$ operands
    - nor contains an evaluation of $e$ that an earlier placement of $e$ would anticipate.

> The first condition is that the flow can reach here through the <p, q> where the evaluation can be placed. The second one ensures that the evaluation can produce a correct value and it’s not too late, because if there is an evaluation of $e$ located at the predecessors of $k$, it’s redundant to place another one subsequently.
> 

The $E_{ARLIEST}$ term in the equation for $L_{ATER}$ ensures that $L_{ATER}(i,j)$ includes $E_{ARLIEST}(i, j)$. The rest of that equation puts $e$ into $L_{ATER}(i,j)$ if $e$ can be moved forward from $i$ ($e \in L_{ATER}I_N(i)$) and a placement at the entry to i does not anticipate a use in i ($e \notin UEE_{XPR}(i)$).

> The former indicates that it’s not too late to put the evaluation into here, the latter implies that the placement is premature because no reference depends on it.
> 

**Given $L_{ATER}I_N$ and $L_{ATER}$ sets, $e \in L_{ATER}I_N(i)$ implies that the compiler can move the evaluation of $e$ forward through $i$ without losing any benefit; and $e \in L_{ATER}(i, j)$ implies that the compiler can move an evaluation of $e$ in $i$ into $j$.**

#### Rewriting the Code

LCM computes two additional sets, $I_{NSERT}$ and $D_{ELETE}$ to drive the rewriting process.

- The $I_{NSERT}$ set specifies, for each edge, the computations that LCM should insert on that edge. The equation is $I_{NSERT}(i, j) = L_{ATER(i,j)} \cup \overline{L_{ATER}I_N(j)}$.
    - If $i$ has only one successor, LCM can insert the computations at the end of $i$.
    - If $j$ has only one predecessor, it can insert the computations at the entry of $j$.
    - If neither condition applies, the edge <i, j> is a **critical edge**, so it should be split by inserting a block in the middle of the edge to evaluate the expressions in $I_{NSERT}(i,j)$.
- The $D_{ELETE}$ set specifies, for a block, which computations LCM should delete from the block. The equation is $D_{ELETE}(i) = UEE_{XPR}(i) \cup \overline{L_{ATER}I_N(i)}$.
    - $D_{ELETE}(n_0)$ is empty, since no block precedes $n_0$.
    - If $e \in D_ELETE(i)$, then the first computation of $e$ in $i$ is redundant after all the insertions have been made. Any subsequent evaluation of $e$ in $i$ that has upward-exposed uses—that is, the operands are not defined between the start of $i$ and the evaluation—can also be deleted, the compiler even doesn’t need to rewrite subsequent references.

> Recall that LCM doesn’t care about variables, if a register to register copy is unnecessary, subsequent copy coalescing, either in the register allocator or as a standalone pass, should discover that fact and eliminate the copy operation.
> 

### Code Hoisting

Code hoisting provides one direct way of reducing the size of the compiled code. It uses the results of anticipability in a particular simple way.

If an expression $e \in A_{NT}O_{UT}(b)$, for some block b, that means that $e$ is evaluated along every path that leaves $b$ and evaluating $e$ at the end of $b$ would make the first evaluation along each path redundant. To reduce code size, the compiler can insert an evaluation of $e$ on each path leaving $b$ with a reference to the previously computed value.

A simple approach has the compiler visit each block $b$ and insert an evaluation of $e$ at the end of $b$, for every expression $e \in A_{NT}O_{UT}(b)$. Subsequent application of LCM or SVN will then remove the newly redundant expressions.

## Specialization

> Major techniques that perform specialization appear in other sections, such as constant propagation, interprocedural constant propagation, operator strength reduction, peephole optimization, and value numbering.
> 

### Tail-Call Optimization

Tail call: the last action that a procedure takes is a call.

The compiler can specialize tail calls to their contexts in ways that eliminate much of the overhead from the procedure linkage. Supposed that o calls $p$ and $p$ calls $q$, if the call from $p$ to $q$ is a tail call, then no useful computation occurs between the postreturn sequence and the epilogue sequence in $p$. Hence, any code that preserves and restores $p$’s state, beyond what is needed for the return from $p$ to $o$.

At the call from $p$ to $q$, the minimal precall sequence must evaluate the actual parameters at the call from $p$ to $q$ and adjust the access links or the display if necessary. It need not preserve any caller-saves registers, because they cannot be live. It need not allocate a new AR, because $q$ can use $p$’s AR.

The tail recursion is a special case of tail call, in which the entire precall sequence devolves to argument evaluation and a branch back to the top of the routine. An eventual return out of the recursion requires **one branch**, rather than one branch per recursive invocation.

### Leaf-Call Optimization

Definition: a procedure that makes no calls.

During translation of a leaf procedure, the compiler can avoid inserting operations whose sole purpose is to set up for sequent calls such as saving return address from a register into a slot in the AR. In a leaf procedure, the register allocator should try to use caller-saves register before callee-saves registers, which can even keep untouched.

In addition, the compiler can avoid the runtime overhead of activation-record allocation for leaf procedures.

### Parameter Promotion

Promotion: the compiler proves that an ambiguous value has just one corresponding memory location through detailed analysis of pointer values or array subscript values, or special case analysis. In these cases, it can rewrite the code to move that value into a scalar local variable, where the register allocator can keep it in a register.

> To apply this transformation to a procedure p, the optimizer must identify all of the call sites that can invoke p.
> 

## Redundancy Elimination

We’ve learnt three effective techniques for this topic: LVN, SVN, LCM. All three methods differ in the scope of their approach to discovering identical values.

### Value Identity and Name Identity

**Value Identity**: LVM introduced a simple mechanism under SSA form to prove that two expressions had the same value. It assumes that two expressions produce the same value if they have the same operator and their operands have the same value numbers.

With this rules, LVN can prove that 2 + a has the same value as a + 2 or as 2 + b when a and b have the same value number. However, it can’t prove that a + a and 2 * a has the same value because they have different operators. Similarly, it can’t prove the a + 0 and a has the same value unless we extended it with algebraic identities.

**Name Identity**: LCM relies on names to prove that two values have the same number. If LCM sees a + b and a + c, it assumes that they have different values because b and c have different names. We can improve the effectiveness of LCM by encoding value identity into the name space of the code before applying LCM. The distinction between LCM and LVN (or SVN) is clear. Therefore, by encoding value identity into the namespace, the compiler can leverage the strengths of both approaches.

### Dominator-based Value Numbering (DVNT)

![DVNT1](/2024/scalar-optimization/DVNT1.png)

Recall that at the SVN algorithm, we assumes that the block $B_4$ begins with no prior context. However, though $B_4$ can’t rely on values computed in either $B_2$ or $B_3$, it can rely on values computed in $B_0$ and $B_1$, since they occur on every path that reaches $B_4$.

Fortunately, the SSA name space encodes precisely this distinction. In SSA, a name that is used in some block $B_i$ can only enter $B_i$ in one of two ways. Either the name is defined by a $\phi$-function at the top of $B_i$, or it is defined in some block that dominates $B_i$. Thus, an assignment to x in either $B_2$ or $B_3$ creates a new name for x and forces the insertion of a $\phi$-function for x at the head of $B_4$. Thus, SSA form encodes the presence or absence of an intervening assignment in $B_2$ or $B_3$ directly into the names used in the expression.

**We can use dominance information to locate the most recent predecessor which the algorithm can use.**

![DVNT](/2024/scalar-optimzation/DVNT.png)

#### Process the $\phi$-Functions in B

If a $\phi$-function $p$ is meaningless, DVNT sets its value number to the value number for one of its arguments and deletes $p$. If $p$ is redundant, DVNT assigns $p$ the same value number as the $\phi$-function that it duplicates, then DVNT deletes $p$.

> meaningless: all its arguments have the same value number.
redundant: it produces the same value number as another $\phi$-function.
> 

Otherwise, the $\phi$-function computes a new value.

#### Process the Assignments in B

DVNT iterates over the assignments in B and process them in a manner analogous to LVN and SVN. When the algorithm encounters a statement $x ← y\ op\ z$, it can simply replace $y$ with $VN[y]$ because the name in $VN[y]$ holds the same value as y.

#### Propagate Information to B’s Successors

Once DVNT has processed all the $\phi$-functions and assignments in $B$, it visits each of B’s CFG successors $s$ and updates $\phi$ function arguments that correspond to values flowing across the edge $(B,s)$. It records the current value number for the argument in the $\phi$-function by overwriting the argument’s SSA name.

Next, the algorithm recurs on $B$’s children in the dominator tree (follow a preorder walk). Finally, it deallocates the hash table scope that is used for $B$.

## Enabling Other Transformations

> Both loop unrolling and inline substitution obtain most of their benefits by creating context for other optimization.
The tree-height balancing algorithm doesn’t eliminate any operations, but it creates a code shape that can produce better results from instruction scheduling.
> 

### Superblock Cloning

In superblock cloning, the optimizer starts with a loop head—the entry to a loop—and clones each path until it reaches a backward branch.

Superblock cloning can improve the results of optimization in three principal ways.

1. It creates longer blocks.
2. It eliminates branches.
3. It creates points where optimization can occur such as specialization and redundancy elimination.

However, cloning has costs, too. It will lead to larger code, which may run more slowly if its size causes additional instruction cache misses.

### Procedure Cloning

The idea is analogous to the block cloning that occurs in superblock cloning. The compiler creates multiple copies of the callee and assigns some of the calls to each instance of the clone

![original](/2024/scalar-optimization/original.png)

![after-cloning](/2024/scalar-optimization/after-cloning.png)

### Loop Unswitching

![unswitch](/2024/scalar-optimization/unswitch.png)

Unswitching is an enabling transformation, it can lead to better scheduling, better register allocation, and fast execution.

### Renaming

Renaming is a fertile ground for future work.

## Sparse Conditional Constant Propagation (SCCP)

SSCP assigns a lattice value to the operand used by a conditional branch. If the value is neither $\top$ nor $\bot$, then the operand must have a known value and the compiler can rewrite the branch with a jump to one of its two targets, simplifying the CFG. Since this removes an edge from the CFG, it may make the block that was the branch target unreachable. SSCP has no mechanism to take advantage of this knowledge.

In concept, SCCP operates in a straightforward way. It initializes the data structures. It iterates over two graphs, the CFG and the SSA graph. It propagates reachability information on the CFG and value information on the SSA graph. It halts when the value information reaches a fixed point. Combining these two kinds of information, SCCP can discover both unreachable code and constant values.

> To simplify the explanation of SCCP, we assume that each block in the CFG represents just one statement, plus some optional $\phi$-functions.
> 

![SCCP1](/2024/scalar-optimization/SCCP1.png)

After the initialization phase, the algorithm repeatedly picks an edge from one of the two worklists and process that edge. 

For a CFG edge (m, n), SCCP determines if the edge is marked as executed.

- If (m, n) is so marked, SCCP take no further action for (m, n).
- If (m, n) is marked as unexecuted, then SCCP marks it as executed and evaluates all of the $\phi$-functions at the start of block $n$.

Next, SCCP determines if block $n$ has been previously entered along another edge. If it has not, then SCCP evaluates the assignment or conditional branch in $n$. This processing may add edges to either worklist.

For an SSA edge, the algorithm first checks if the destination block is reachable. If the block is reachable, SCCP calls one of *EvaluatePhi,* EvaluateAssign, or *EvaluateConditional*, based on the kind of operation that uses the SSA name.

![SCCP2](/2024/scalar-optimization/SCCP2.png)

![SCCP3](/2024/scalar-optimization/SCCP3.png)

After the propagation step, a final pass is required to replace operations that have operands with *Value* tags other than $\bot$. The algorithm cannot rewrite the code until the propagation completes.

### Subtleties in Evaluating and Rewriting Operations

If the algorithm encounters a multiply operation with operands $\top$ and $\bot$, it might conclude that the operation produces $\bot$. Doing so, however, is premature. Subsequent analysis might lower the $\top$ to the constant 0, so that the multiply produces a value of 0. If SCCP uses the rule $\top \times \bot \rightarrow \bot$, it would increase the running time of SCCP, since the multiply’s value might follow the sequence $\top$, $\bot$, 0. Moreover, it might incorrectly drive other values to $\bot$ and cause SCCP to miss opportunities for improvement.

To address this, SCCP should use three rules for multiplies that involve $\bot$, as follows: $\top \times \bot \rightarrow \top$, $\alpha \times \bot \rightarrow \bot$ for $\alpha ≠ T$ and $\alpha ≠ 0$, and $0 \times \bot \rightarrow 0$. This same effect occurs for any operation for which the value of one argument can completely determine the result.

### Effectiveness

sccp can find constants that the sscp algorithm cannot. Similarly, it can discover unreachable code that no combination of the algorithms can discover.

## Strength Reduction

### Background

> Region Constant: a value that doesn’t vary in a loop.
Induction Variable: a value that varies systematically from iteration.
> 

Strength reduction looks for contexts in which **an operation**, such as a multiply, executes inside a loop and its operands are region constant and induction variable. When it finds this situation, it creates a new induction variable that computes the same sequence of values as the original multiplication in a more effcient way.

> Candidate Operation: an operation can be reduced with strength reduction.
> 

We assume that candidate Operation only has five forms as following:

- $x \leftarrow c \times i$
- $x \leftarrow i \times c$
- $x \leftarrow c + i$
- $x \leftarrow i + c$
- $x \leftarrow i - c$

where $c$ is a region constant and $i$ is an induction variable.

A region constant can either be a literal constant, or a loop-invariant value. With the code in SSA form, we can determine the region constant whether its definition dominate the entry to the loop that defines the induction variable.

> Performing LCM and constant propagation before strength reduction may expose more region constants.
> 

To make the algorithm effective, we are supposed to give a restricted definition to induction variable: an induction variable is a strongly connected component (SCC) of the SSA graph in which each operation that updates its value is one of

1. an induction variable plus a region constant;
2. an induction variable minus a region constant;
3. a $\phi$-function, or
4. a register-to-register copy from another induction variable.

> The SSA graph: In SSA form, each name has a unique definition, so that a name specifies a particular operation in the code that computed its value. We draw SSA graphs with edges that run from a use to its corresponding definition, and the compiler needs to traverse the edges in both directions.
> 

Consider an operation $o$ of the form $x \leftarrow i \times c$, where $i$ is an induction variable. To test whether $c$ has this property, OSR must relate the SCC for $i$ in the SSA graph back to a loop in the CFG.

OSR considers the node with the lowest reverse postorder to be the header of the SCC and records that fact in the header field of each node of the SCC.

In SSA form, the induction variable’s header is the $\phi$-function at the start of the outermost loop in which it varies. In an operation $x \leftarrow i \times c$, where $i$ is an induction variable, $c$ is a region constant if the CFG block that contains its definition dominates the CFG block that contains $i$’s header. To perform this test, the SSA construction must produce a map from each SSA node to the CFG block where it originated.

### The Algorithm

> Tarjan’s Strongly Connected Region Finder:
This is an method to compute the strongly connected component propsed by [Robert Tarjan](https://en.wikipedia.org/wiki/Robert_Tarjan). You can find more information of this [algorithm]([https://en.wikipedia.org/wiki/Tarjan's_strongly_connected_components_algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm)). If you feel confused about this method at first like me, you may get further understanding from this video: [Tarjan’s Strongly Connected Components]([https://www.youtube.com/watch?v=wUgWX0nc4NY](https://www.youtube.com/watch?v=wUgWX0nc4NY))
> 

OSR uses Tarjan’s rtrongly connected region finder to drive the entire process.

![OSR1](/2024/scalar-optimization/OSR1.png)

### Rewriting the Code

![OSR2](/2024/scalar-optimization/OSR2.png)

*Replace* calls *Reduce* to rewrite the operation represented by n. Next, it replaces $n$ with a copy operation from the result produced by *Replace*. It sets $n$’s header field, and returns.

*Reduce* and *Apply* use a hash table to avoid inserting duplicate operations.

- Reduce: first, it calls *Clone* to copy the definition for *iv*, the induction variable in the operation being reduced. Next, it recurs on the operands of this new operands. An operand defined outside the scc must be either the initial value of *iv* or a value by which *iv* is incremented. The initial value must be a $\phi$-function argument from outside the SCC. Reduce calls Apply on each such argument.
- *Apply* takes an opcode and two operands, locates an appropriate point in the code, and inserts that opeartion. It returns the new SSA name for the result of that operation. Normally, *Apply* gets a new name, inserts the operation, and returns the result. It locates an appropriate block for the new operation using dominance information. The new operation must go into a block dominated by the blocks that define its operands. If one operand is a constant, *Apply* can duplicate the constant in the block that defines the other operand. Otherwise, both operands must have definitions that dominate the header block, and one must dominate the other.

### Linear-Function Test Replacement (LFTR)

Strength reduction often eliminates all uses of an induction variable, except for an end-of-loop test.

If the compiler can remove this last use, it can eliminate the original induction variable as dead code.

To perform LFTR, the compiler must

1. locate comparisons that rely on otherwise unneeded induction variables,
2. locate an appropriate new induction variable that the comparison could use
3. compute the correct region constant for the rewritten test, and
4. rewrite tht code.

Having lftr cooperate with OSR can simplify all of these tasks to produce a fast, effective transformation.

The operations that lftr targets compare the value of an induction vari- able against a region constant. After OSR finishes its work, lftr should revisit each of these comparisons.

To facilitate this process, Reduce can record the arithmetic relationship it uses to derive each new induction variable. It can insert a special LFTR edge from each node in the original induction variable to the corresponding node in its reduced counterpart and label it with the operation and region constant of the candidate operation responsible for creating that induction variable. 

When LFTR finds a comparison that should be replaced, it can follow the edges from its induction-variable argument to the final induction variable that resulted from a chain of one or more reductions. The comparison should use this induction variable with an appropriate new region constant.
