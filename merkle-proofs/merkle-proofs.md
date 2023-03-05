

# Intro

Recently I started working with the TON blockchain and its lite server API specifically. 
Multiple API methods of the TON runtime return proofs.    

For example, `getOneTransaction` method looks as follows:    
```tl
liteServer.getOneTransaction#ea240fd4 id:tonNode.blockIdExt account:liteServer.accountId lt:long = liteServer.TransactionInfo;
```
and it returns a TransactionInfo instance containing a "proof":
```tl
liteServer.transactionInfo id:tonNode.blockIdExt proof:bytes transaction:bytes = liteServer.TransactionInfo;
```

The proof is a way to verify that a requested transaction is included in a particular block.
This is a kind of basic expectation. But what is the proof exactly? 

I didn't find a good explanation of what the proof is and 
decided to write this post for those who just started their journey with TON.
The post shows how the proof is constructed and how different types of exotic cells are used. 

# Short introduction of TL-B and the Bag of Cells philosophy

TON introduces a new concept to work with data known as the Bag of Cells philosophy.
A single cell contains up to 1023 bits of data and up to 4 references to other cells.
So it is possible to represent any complex data type as a tree of cells.
The TL-B schema defines rules for doing that.

Going back to the `getOneTransaction` method. A block is at the core of the TON blockchain. 
Among other things a block contains a list of transactions.
This means that a block internally is represented as a tree of cells. 
Somewhere inside a block's tree, there are cells which our transaction consists of. 
The idea is that TON is not going to send the whole block. 
Instead, it sends a block's tree processed in such a way that  
the client can verify from their side that our transaction is in that tree.

# TON version of Merkle Tree

[The standard Merkle Tree](https://en.wikipedia.org/wiki/Merkle_tree) consists of inner nodes and leaves. 
Only leaves contain data. Inner nodes farther up the tree are hashes of their respective children.

The TON Merkle Tree is a bit different in a way that any node in a tree is a cell containing useful data.

| An example of a cell tree [Figure 0] |
|--------------------------------------|
| ![A tree of cell](images/0.svg)      |

Let's take a brief look at how hashes are calculated:

H<sub>0</sub>(1) = SHA256(DATA(1) + H<sub>0</sub>(2) + H<sub>0</sub>(3) + H<sub>0</sub>(4) + DEPTH(2) + DEPTH(3) + DEPTH(4))      
H<sub>0</sub>(2) = SHA256(DATA(2) + H<sub>0</sub>(5) + H<sub>0</sub>(6) + DEPTH(5) + DEPTH(6))   
H<sub>0</sub>(3) = SHA256(DATA(3) + H<sub>0</sub>(7) + H<sub>0</sub>(8) + DEPTH(7) + DEPTH(8))   
H<sub>0</sub>(4) = SHA256(DATA(4) + H<sub>0</sub>(9) + DEPTH(9))   
H<sub>0</sub>(5) = SHA256(DATA(5))   
... and so on ...

Let's ignore subscript 0 for now, we'll talk about what it means later. 

**DATA(X)** represents useful data inside X cell.   
**DEPTH(X)** stands for the standard height of a tree: the number of edges between the lowest leaf in X's subtree and X itself.
Here, DEPTH(14)=0, DEPTH(9)=1, DEPTH(4)=2, DEPTH(1)=3. 
Depth is included in a hash to avoid [a second-preimage attack](https://en.wikipedia.org/wiki/Merkle_tree#Second_preimage_attack).    

Given this definition of a hash, you can see that 
if a single bit is changed in 14, its hash will also change, as well as hashes of 9, 4 and eventually 1.

**So a root's hash of a cell tree is a commitment to the data in the tree.**

# First example: algorithm to construct a Merkle proof 

The first example is quite simple. 
We will be going through several steps to construct a Merkle proof 
and introduce two special types of cells: Merkle proof and pruned branch.

Ok, we are given a tree in Figure 1.1, and we want to create a Merkle proof that 7, 10 and 11 are in the tree.
Let's go through the following steps:   
* color 7, 10 and 11 in red.    
* color the path from the red root 7 up to the root of the tree (1 and 3) in red.   
* get rid of 2, 6 and their subtrees. But because 2 and its subtree are used to
  calculate H<sub>0</sub>(1), we replace 2 with a special cell type
  known as **a pruned branch cell**. 
  A pruned branch cell replaces a subtree and embeds a hash of the subtree's root and its depth into the DATA section of the cell.
  In our case: 
     * the DATA section of P2 contains H<sub>0</sub>(2) and DEPTH(2). 2 and its subtree have gone.
     * the DATA section of P6 contains H<sub>0</sub>(6) and DEPTH(6). 6 and its subtree have also gone.
  The result is shown in Figure 1.3.
  According to our definition of H:   
  **H<sub>0</sub>(1) = SHA256(DATA(1) + H<sub>0</sub>(2) + H<sub>0</sub>(3) + DEPTH(2) + DEPTH(3))**   
  as you can see, 2 is not present in the tree anymore but H<sub>0</sub>(2) and DEPTH(2) can be taken from P2, 
  meaning it is still possible to calculate H<sub>0</sub>(1).
  The only reason of having a pruned branch cell is to keep the hash and depth of a removed subtree around. 

* attach the pruned tree to a newly created cell, known as **a Merkle proof cell** (Figure 1.4).

| [Figure 1.1] Proving red cells are in a tree | [Figure 1.2] Adding a path from 7 to 1 |
|----------------------------------------------|----------------------------------------|
| ![red cells in tree](images/1-1.svg )        | ![adding path to root](images/1-2.svg) |


| [Figure 1.3] Pruning subtrees of 2 and 6 | [Figure 1.4] A proof     |
|------------------------------------------|--------------------------|
| ![pruning subtrees](images/1-3.svg )     | ![proof](images/1-4.svg) | 

That's it. The proof has been constructed.

So what are we going to do with this proof? 
Imagine, the original tree in Figure 1.1 is [a block](https://github.com/ton-blockchain/ton/blob/v2023.01/crypto/block/block.tlb#L446-L449)
and 7, 10, 11 represent a transaction inside that block.
We can serialize the proof in Figure 1.4 and send it as a response to `getOneTransaction`, which will be `proof:bytes` of TransactionInfo:
```tl
liteServer.transactionInfo id:tonNode.blockIdExt proof:bytes transaction:bytes = liteServer.TransactionInfo;
```

Then, when we get this proof from the TON's lite server:     
* We can deserialize it using TL-B schema, and we will get a tree.    
* The root of this tree is a Merkle proof cell.   
* A single reference of the Merkle proof cell is a pruned tree of a block in which our transaction is included.    
* A hash inside the Merkle proof cell must match the block's root hash.   
* The whole transaction is included in the proof.    
* We can carefully extract the transaction from the proof.   

# Second example: nested Merkle proofs 

A Merkle proof can be included as a subtree inside another tree of cells.   
Figure 2.1 shows an example of A which references two cells: B and MP from Figure 1.4.
In this example, we construct a new proof of inclusion of MP and 1.
Let's perform the same steps:
* adding a path from MP to A (Figure 2.2).
* pruning B and 3 and their subtrees (Figure 2.3).
* attaching the pruned tree to a new Merkle proof cell (Figure 2.4).

| [Figure 2.1] Tree of cells | [Figure 2.2] Adding path from MP to A  |
|----------------------------|----------------------------------------|
| ![tree](images/2-1.svg )   | ![adding path to root](images/2-2.svg) |


| [Figure 2.3]  Pruning subtrees of B and 3 | [Figure 2.4] Merkle proof |
|-------------------------------------------|---------------------------|
| ![pruning subtrees](images/2-3.svg)       | ![proof](images/2-4.svg)  | 

# The use of levels and higher level hashes

So far we have been working with H<sub>0</sub>. 
We defined H<sub>0</sub> of a root cell as a commitment to the data in the whole tree. 
This works perfectly well for ordinary non-pruned trees. But 
our definition of a hash treats a pruned branch cell as a container holding a hash of a removed subtree.
The goal of a level is 
to extend the definition to include all pruning modifications of a tree in a commitment.

## Rules to assign a level 

A level is a property of a cell. It is an integer that lies in an interval [0..3]. 
Each cell type has its own rules to assign a level.
These rules form a strict system, where levels of a cell and its references depend on each other. 
This post doesn't describe the system in detail, as it would require a deep dive into implementation.
Rather the post aims to develop a basic intuition about TON's Merkle proofs.

Let's take the tree from the previous example and figure out the level of every cell in the tree.
The TON whitepaper is a good place to start:
```
The level l of a pruned branch cell may be called its de
Brujn index, because it determines the outer Merkle proof or Merkle
update during the construction of which the branch has been pruned.
```
So the level of a pruned branch cell shows which Merkle proof cell this pruned branch cell was created for.

[Figure 3.1] If we start from P2 and go up to the root of the tree,
Level(P2)=1 means that P2 is related to the first MP we meet on our path.
Level(P3)=2 means that P3 is related to the second MP.

A full set of rules looks as follows:
* pruned branch cell: a level determines the outer MP this PB cell is related to.
* ordinary cell: a level of an ordinary cell equals to the max level of all its references.
* Merkle proof cell:  a level of a Merkle proof cell equals to the level of its single reference decreased by 1 but not less than zero.

[Figure 3.2] Applying these rules to the tree in Figure 3.1, we get:
* Level(1) = max(Level(P2), Level(P3)) = 2     
* Level(Red MP) = Level(1) - 1 = 1    
* Level(A) = max(Level(PB), Level(Red MP)) = 1    
* Level(Gray MP) = Level(A) - 1 = 0.    

| Levels of pruned branch cells [Figure 3.1]             | Levels of all cells in a tree [Figure 3.2] |
|--------------------------------------------------------|--------------------------------------------|
| ![Tree with Pruned Branch cell levels](images/3-1.svg) | ![Tree with cell levels](images/3-2.svg)   | 

## Different types of hashes 

It's time to get back to our definition of a hash. 
H<sub>0</sub>(X) stands for a hash of level 0 of a cell X. Given that a level lies in an interval [0..3],
we can say that any cell in a tree has four hashes. 
Don't be confused, a cell still has exactly one level assigned (from now on let's call it the actual level of a cell), 
but at the same time it has four hashes. 
Now, we will figure out how the hashes are calculated.
As a general rule, a cell's hashes of levels above the actual level of the cell equal to the actual level's hash.

Each time we prune a tree to get a new Merkle proof, 
we modify the tree and in some sense we create a new version of the tree. 
Let's take a look at what happens to root cell 1 from our examples.

| [Figure 4.1] Original tree       | [Figure 4.2] Pruning tree first time | [Figure 4.3] Pruning tree second time | 
|----------------------------------|--------------------------------------|---------------------------------------|
| ![original tree](images/4-1.svg) | ![original tree](images/4-2.svg)     | ![original tree](images/4-3.svg)      |

[Figure 4.1] This is the original tree we started from. 
No exotic cells, all cells are ordinary. All cells have level 0 including root cell 1.
For any cell in the original tree: H<sub>0</sub>(X) = H<sub>1</sub>(X) = H<sub>2</sub>(X) = H<sub>3</sub>(X).

[Figure 4.2] Then we prune the tree, 
cut off some of its subtrees and introduce the second version of the same tree, slightly modified. 
Now Level(1)=1. Nothing has changed regarding H<sub>0</sub>(1).
To calculate H<sub>0</sub>(1), we take H<sub>0</sub>(2) and H<sub>0</sub>(6) from DATA sections of P2 and P6 correspondingly.
H<sub>0</sub>(1) is still the same, it is still a commitment to the data in the original tree.

But what is H<sub>1</sub>(1)?

Higher hashes are computed a bit differently:   
H<sub>1</sub>(1) = SHA256(H<sub>0</sub>(1) + H<sub>1</sub>(P2) + H<sub>1</sub>(3) + DEPTH(P2) + DEPTH(3))      
H<sub>1</sub>(P2) = SHA256(DATA(P2))   
H<sub>1</sub>(3) = SHA256(H<sub>0</sub>(3) + H<sub>1</sub>(P6) + H<sub>1</sub>(7) + DEPTH(P6) + DEPTH(7))   
...

Two things here:
* instead of using DATA(x) as the first element for ordinary cells in their cell representations, we use a hash of the previous level:
  **H<sub>1</sub>(1) = SHA256(H<sub>0</sub>(1) + ... )**.  
* a hash of a pruned branch cell is defined as follows:
  for levels below the actual level of a pruned branch cell, we extract hashes from the cell itself, 
  but for the actual level's hash we use data of the cell:
  H<sub>0</sub>(P2) = H<sub>0</sub>(2), but H<sub>1</sub>(P2) = SHA256(DATA(P2)).

So, it is possible to think of H<sub>1</sub>(1) as a commitment to both
the data in the original tree and the second version of the tree. 
This is because changing a bit in the original tree will change H<sub>0</sub>(1), which will in turn change H<sub>1</sub>(1).

[Figure 4.3] When we further modify the tree, we get the third version of it.   
H<sub>0</sub>(1) in Figure 4.3 equals to H<sub>0</sub>(1) in Figure 4.1,   
H<sub>1</sub>(1) in Figure 4.3 equals to H<sub>1</sub>(1) in Figure 4.2,  
H<sub>2</sub>(1) = SHA256(H<sub>1</sub>(1) + H<sub>2</sub>(P2) + H<sub>2</sub>(P3) + DEPTH(P2) + DEPTH(P3)).   

Hashes H<sub>2</sub>(1), H<sub>1</sub>(1), H<sub>0</sub>(1) form a dependency chain, where 
H<sub>2</sub>(1) depends on H<sub>1</sub>(1) and H<sub>1</sub>(1) depends on H<sub>0</sub>(1).
Such a chain makes H<sub>2</sub>(1) a commitment to the data in the original tree as well as to the second and third versions of the tree.

| Hashes used to calculate H<sub>0</sub>(Gray MP) [Figure 5] |
|------------------------------------------------------------|
| ![A tree of cell](images/5-1.svg)                          |

The last ingredient that is missing is a hash of a Merkle proof cell.
**H<sub>0</sub>**(Gray MP) = SHA256(DATA(Gray MP) + **H<sub>1</sub>(A)** + DEPTH(A)).   
We don't use H<sub>0</sub>(A), but instead we use a hash of the level of MP incremented by 1: H<sub>1</sub>(A).  
The same happens to the red MP:   
**H<sub>1</sub>**(Red MP) = SHA256(DATA(Red MP) + **H<sub>2</sub>(1)** + DEPTH(1)).   

As you can see H<sub>0</sub>(Gray MP) depends on the hashes of all cells with levels above or equal to the actual levels of the cells.
H<sub>0</sub>(Gray MP) is a commitment to the whole tree including all modifications.

That's it. 