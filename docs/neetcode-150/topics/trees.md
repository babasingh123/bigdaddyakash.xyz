# Trees

> **15 problems** · prerequisites below

## What this topic teaches

Tree problems are recursion problems in costume. The single most important habit this topic builds is the **"what should this function return for a single subtree?"** mindset: define a recursive contract precisely (often returning a tuple — `(height, is_balanced)`, `(max_path_through, max_path_overall)`), trust the recursive calls on left and right, and combine. Once the contract is right, the code is usually 6–10 lines.

The second skill is choosing the right traversal: **DFS** (pre/in/post-order) for "compute something dependent on the entire subtree," **BFS / level-order** for "do something per layer" (right-side view, zigzag, level averages). BSTs add a third tool: the **in-order traversal of a BST is sorted**, which unlocks *Validate BST*, *Kth Smallest*, and *LCA of BST* in clean ways. Hard problems in this topic usually combine two of these: *Max Path Sum* is "post-order DFS with a tuple-style return," *Serialize/Deserialize* is "pre-order DFS with explicit nulls."

## Prerequisites

- BST Insert and Remove (Data Structures & Algorithms for Beginners)
- Depth-First Search (Data Structures & Algorithms for Beginners)
- Breadth-First Search (Data Structures & Algorithms for Beginners)
- BST Sets and Maps (Data Structures & Algorithms for Beginners)
- Iterative DFS (Advanced Algorithms)

## Core patterns

### Pattern 1: Post-order DFS with a tuple return

For each node, the recursive call returns "the information I need about the subtree rooted here." *Balanced Binary Tree* returns `(height, is_balanced)`; *Diameter of Binary Tree* returns the height while updating a global `best`; *Binary Tree Maximum Path Sum* returns the best **straight** path through the node while updating a global with the best **bent** path. The pattern is always: recurse left, recurse right, combine, return.

### Pattern 2: Mirror / two-tree DFS

When the problem compares two trees structurally (*Same Tree*, *Subtree of Another Tree*, *Invert Binary Tree*), recurse on a pair of nodes (or one and its mirror). Base cases: both null → equal; one null → unequal. *Invert Binary Tree* is the same shape but mutating: swap children, recurse.

### Pattern 3: Level-order BFS

A queue (`collections.deque`), one node per slot. Process level-by-level by snapshotting `len(queue)` at the top of each iteration — that's exactly the level's size. *Binary Tree Level Order Traversal* (groups), *Right Side View* (take the last node of each level), *Count Good Nodes* (BFS or DFS, carrying max-so-far as state).

### Pattern 4: BST exploit — in-order is sorted

*Validate BST* compares each node to a running `(low, high)` interval (recursive) or to the previous in-order node (iterative). *Kth Smallest Element In a BST* runs in-order traversal and decrements `k`. *Lowest Common Ancestor of a BST*: walk down from root, go left if both `p` and `q` are smaller, right if both larger, otherwise the split point is the LCA.

### Pattern 5: Reconstruct from traversals

*Construct Binary Tree From Preorder And Inorder Traversal*: the first element of preorder is the root; find it in inorder to split left/right subtrees; recurse. Use a hash map from value → inorder index to make each split O(1), giving O(n) overall.

### Pattern 6: Serialize / deserialize

*Serialize And Deserialize Binary Tree* uses **pre-order DFS with null markers**: emit `val, #` for missing children. To deserialize, consume tokens left-to-right; on `#` return `None`, otherwise build a node and recurse on left then right. The encoding's uniqueness is what makes the reconstruction unambiguous without inorder.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Invert Binary Tree | Easy |
| 2 | Maximum Depth of Binary Tree | Easy |
| 3 | Diameter of Binary Tree | Easy |
| 4 | Balanced Binary Tree | Easy |
| 5 | Same Tree | Easy |
| 6 | Subtree of Another Tree | Easy |
| 7 | Lowest Common Ancestor of a Binary Search Tree | Medium |
| 8 | Binary Tree Level Order Traversal | Medium |
| 9 | Binary Tree Right Side View | Medium |
| 10 | Count Good Nodes In Binary Tree | Medium |
| 11 | Validate Binary Search Tree | Medium |
| 12 | Kth Smallest Element In a BST | Medium |
| 13 | Construct Binary Tree From Preorder And Inorder Traversal | Medium |
| 14 | Binary Tree Maximum Path Sum | Hard |
| 15 | Serialize And Deserialize Binary Tree | Hard |

## Tips

- **Write the recursive contract as a one-line docstring before the body.** "Returns `(height, is_balanced)`" makes the rest of the function self-writing.
- **For *Validate BST*, pass the `(low, high)` interval down**, don't try to validate "left subtree is less than root" locally — that's the canonical wrong answer for trees like `[5, 1, 4, null, null, 3, 6]`.
- **For *Max Path Sum*, return the best "straight" path through the node, update the answer with the best "bent" path.** Clamp negative subtree returns to `0` (you can always choose not to extend into a negative-sum subtree).
- **Don't confuse "depth" and "height."** Depth is from the root down (BFS counts it naturally); height is from the leaves up (post-order returns it).
- **Iterative DFS exists.** When recursion depth is a concern (very unbalanced trees, ~10⁵ nodes), convert to explicit stack — the shape mirrors recursive code one-to-one.

## Linked concepts

Trees themselves rarely show up directly in the design problems on this site — but **BSTs are the conceptual basis** for ordered structures like sorted maps and segment trees, and the recursive DFS pattern is the same one that powers:

- [Add Search Words (DSA Design)](../../dsa-design/problems/add-search-words.md) — recursive DFS over a tree (a Trie, in this case).
- [Word Dictionary (DSA Design)](../../dsa-design/problems/word-dictionary.md) — same recursive DFS pattern, with wildcard branching.
- [Range Sum Query (DSA Design)](../../dsa-design/problems/range-sum-query.md) — segment trees, the BST's cousin for range queries.
