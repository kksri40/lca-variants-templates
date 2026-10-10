# lowest common ancestor: what LCA actually means, plus templates for the binary tree, BST, parent-pointer, and 100k-query versions

Open a terminal on any repo with two long-lived branches and run `git merge-base main feature`. You just got a lowest common ancestor answer: the deepest commit that is an ancestor of both. Git needs that node constantly to figure out where two lines of history diverged, which is why the question shows up in file systems, DOM trees, phylogenetic trees, and roughly every tree-shaped interview problem you'll be asked.

It's also one of those topics where people memorize five separate solutions and then freeze when the problem statement changes one detail. The variants are real, but they come from the same definition. Get the definition straight and the code mostly writes itself.

## The definition, stated precisely enough to code from

The lowest common ancestor of two nodes `p` and `q` in a rooted tree is:

> The deepest node that has both `p` and `q` as descendants, where a node counts as a descendant of itself.

That last clause is where most bugs come from. If `p` is an ancestor of `q`, then `p` is the LCA, even though `p` looks "too high" if you're picturing the answer as a fork. Take a tree where `3` is the root, `5` and `1` sit under it, and `4` hangs off `5`. The LCA of `5` and `4` is `5`, not `3`. The LCA of `5` and `1` is `3`.

"Lowest" here means deepest, not smallest by value. That phrasing has confused enough people that the second thing worth internalizing is the split-point intuition: walk up from `p` and up from `q`; the LCA is the first node where the two upward paths meet, which is also the first node on the way down where `p` and `q` land in different subtrees (or where one of them *is* the current node).

If you want a five-second sanity check on your understanding, this identity from competitive programming is a good one:


distance(u, v) = depth[u] + depth[v] - 2 * depth[lca(u, v)]


If you can't compute your LCA, you can't compute distances in a tree — which is why LCA shows up as a sub-problem inside harder questions rather than as a standalone puzzle.

## The five variants you'll actually meet

LeetCode has a small family of these, and the differences are exactly the details candidates miss.

| Problem | Setup | The twist | Target complexity |
| --- | --- | --- | --- |
| 236 — LCA of a Binary Tree | Plain binary tree, both nodes guaranteed present | Node references, not values | O(n) time, O(h) space |
| 235 — LCA of a BST | Binary **search** tree | Left/right ordering lets you skip whole subtrees | O(h) time, O(1) space |
| 1650 — LCA III | Nodes carry a `parent` pointer, no root given | It's a linked-list intersection problem in disguise | O(h) time |
| 1644 — LCA II | `p` or `q` may not exist in the tree | You must search the *entire* tree before returning | O(n) |
| 1676 — LCA IV | You're given a list of `k` nodes | Same recursion, membership check is now a set | O(n) |

The classic interview mistake is answering 235 with the 236 solution. It passes, and it throws away the property that makes the problem easy. Interviewers notice.

## The one template behind 236, 1644, and 1676

Everything in the plain-binary-tree family is post-order DFS that answers one question per subtree: *did I find `p`, `q`, or the LCA down here?*

python
def lowestCommonAncestor(root, p, q):
    if not root or root is p or root is q:
        return root
    left  = lowestCommonAncestor(root.left,  p, q)
    right = lowestCommonAncestor(root.right, p, q)
    if left and right:
        return root        # found one target on each side → this is the split point
    return left or right   # both targets in one subtree, or just one found so far


Two things make it work. If `p` and `q` come back from different children, the current node is the first split point, so it's the answer. If only one side returns something non-null, that means either both targets are in that subtree, or the node returned *is* one of the targets and the other lies beneath it — either way, pass it up.

For 1676, swap the two identity checks for `if root.val in targets` with a set built once up front. For 1644, the same skeleton breaks, and it's worth understanding why: the early `return root` when you hit `p` or `q` is only safe because the problem guarantees both nodes exist. If they might not, you have to search everything and record whether you actually saw each one, then return `null` when either is missing.

An alternative worth knowing because it's easier to explain out loud: record the root-to-`p` path and the root-to-`q` path, then return the last node the two paths share. It's O(n) time and O(h) space too, just with more bookkeeping. Good for a whiteboard explanation, slightly clunky to write.

## BST version: stop traversing, start steering

In a BST you never need to look at both subtrees. Compare values and take one branch:

python
def lowestCommonAncestorBST(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left          # both targets are smaller
        elif p.val > root.val and q.val > root.val:
            root = root.right         # both targets are larger
        else:
            return root               # values split here, or root is p or q
    return None


The `else` branch covers three cases at once: `root` sits between the two values, or equals one of them. All three mean you're done. Time is O(h) — O(log n) for a balanced tree, O(n) for a degenerate one — and space is O(1) with the iterative version, which is easier to defend under interview pressure than recursion.

## Parent pointers (1650): two pointers, one loop

Once each node knows its parent, the root node isn't even an input. Walk upward from both nodes and switch to the other starting node when you run off the top:

python
def lowestCommonAncestor(p, q):
    a, b = p, q
    while a is not b:
        a = a.parent if a else q
        b = b.parent if b else p
    return a


This is the same trick as finding the intersection of two linked lists: both pointers travel the same total distance, so they must meet at the first shared node. The alternative — dump all of `p`'s ancestors into a set and climb from `q` — is easier to reason about and costs O(h) extra space.

## When the tree is big and the queries are many

None of the above scales to 10⁵ nodes with 10⁵ queries, because O(h) per query degenerates to O(n) per query on a path-shaped tree. This is where the competitive-programming machinery kicks in, and it's also where an interview question like "what if you needed to answer a million of these?" goes.

**Binary lifting** precomputes `up[v][j]`, the 2ʲ-th ancestor of `v`, so you can jump instead of walk:

python
LOG = 17                          # 2^17 > 1e5
up = [[0] * LOG for _ in range(n + 1)]

# DFS from the root: up[v][0] = parent, and record depth[v]
for j in range(1, LOG):
    for v in range(1, n + 1):
        up[v][j] = up[up[v][j - 1]][j - 1]

def lca(u, v):
    if depth[u] < depth[v]:
        u, v = v, u
    diff = depth[u] - depth[v]
    for j in range(LOG):          # lift the deeper node to the same depth
        if diff >> j & 1:
            u = up[u][j]
    if u == v:
        return u
    for j in range(LOG - 1, -1, -1):   # jump both up while ancestors differ
        if up[u][j] != up[v][j]:
            u, v = up[u][j], up[v][j]
    return up[u][0]


Preprocessing is O(n log n) time and space; each query is O(log n). The second loop is the part people get wrong: you walk down the powers of two from the largest and jump only when the 2ʲ-th ancestors are *different*, which leaves both pointers one level below the LCA.

The other standard approach is the Euler tour plus range-minimum query. Flatten the tree into an Euler tour, store each node's depth alongside it, and the LCA of `u` and `v` becomes the shallowest node appearing between their positions in that array — a sparse table over the depth array answers it in O(1) after O(n log n) preprocessing. There's also Tarjan's offline algorithm, which answers a whole batch of queries with union-find in near-linear time. Unless you're doing competitive programming, binary lifting is the one worth knowing; the Euler tour trick matters more for string algorithms, where LCA queries on suffix trees come up constantly.

| Approach | Preprocessing | Per query | When it's the right call |
| --- | --- | --- | --- |
| Climb parents | O(1) | O(h) | One-off queries, tiny trees |
| DFS on binary tree (236 style) | none | O(n) | Single pair, interview answer |
| BST guided search | none | O(h) | You noticed it's a BST |
| Binary lifting | O(n log n) | O(log n) | Static tree, many queries |
| Euler tour + sparse table | O(n log n) | O(1) | Query-heavy, RMQ already in your toolkit |
| Tarjan offline | ~O(n + q) | amortized | You know all the queries up front |

## Where LCA shows up outside interviews

Three places worth being able to name, because they make the problem feel less academic:

- **Version control.** Git's merge base is the LCA of two commits in the ancestry DAG. Branch divergence points are LCA queries.
- **File systems and DOM.** "Deepest shared directory" for two paths, or the nearest common ancestor element for two DOM nodes — the same traversal you'd write for 236, just without a tidy binary constraint.
- **Biology and string algorithms.** Phylogenetic trees use LCA to find the most recent common ancestor of two species; suffix trees use it to find the longest common prefix of two suffixes, which powers a pile of substring algorithms.

## How to actually drill this instead of grinding 300 random problems

The LCA family rewards pattern recognition more than raw volume. If you solve 236 in isolation, then meet 1650 and reach for recursion, the practice didn't transfer. The useful mental index looks like this:

1. Plain binary tree, both nodes guaranteed → post-order DFS, three lines.
2. Same, but nodes might be missing → full search plus two flags.
3. A list of nodes instead of two → set membership.
4. BST → steer by value comparison, don't traverse.
5. Parent pointers → two-pointer linked-list intersection.
6. Huge static tree, many queries → binary lifting.

Six boxes. Most people have one, and the gap shows up the moment the interviewer changes a sentence in the prompt.

This is the specific thing AlgoMonster organizes its curriculum around, which is why it's a reasonable stop if LCA is the kind of problem that keeps surprising you. The platform was built by ex-Google engineers and structured by pattern rather than by problem list, so the site's LCA material sits inside its tree-traversal pattern family — one lesson under Depth First Search / Advanced for the general binary tree case, another under Depth First Search / Binary Search Tree for the 235 version. The site also publishes free per-problem editorial pages with step-by-step walkthroughs, including ones on LCA problems, so you can see how a solution is explained before paying for anything.

👉 [See the full pattern-based course and free lesson previews](https://algo.monster?fpr=li69)

If you're weighing whether to pay, here's what the checkout page currently shows. Note that AlgoMonster runs constant sales and the numbers on the page move with them — one cached copy listed $69.99/year and $89 lifetime, while third-party blogs quote everything from $89/year to $459 lifetime. Those blog numbers are almost all stale. Check the live page.

| Plan | What's included | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems. Unlimited AI assistant, unlimited Speedrun, real company questions updated monthly, 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, member discount on live bootcamps | $99 (listed as $8.25/mo) | Billed annually, 12 months of access | [See the yearly plan](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates | $189 | One-time payment | [See the lifetime plan](https://algo.monster?fpr=li69) |

Two things to note before you decide. There's no monthly plan — it's annual or lifetime, nothing in between — and third-party reviews consistently say there's no team or enterprise tier, so this is priced for individuals. Also, the format is text, diagrams, and code in multiple languages; there are no video walkthroughs, which is either a feature or a dealbreaker depending on how you learn.

The budgeting logic is straightforward. At $99/year versus $189 once, the lifetime plan pays for itself if you'll still be studying or interviewing two years out. If you have one loop in the next three months and you're already comfortable with trees, the yearly plan is the safer spend — the platform isn't cheap per problem, and its value is the pattern framework, not a large problem bank. If LCA specifically is the thing tripping you up, the free editorial pages cover enough ground to test the teaching style before you commit a dollar.

## Bottom line

Lowest common ancestor isn't five problems. It's one definition — deepest node with both targets as descendants, self included — and a small set of recognitions: does this tree have ordering, do the nodes know their parents, do the targets both exist, and how many queries am I answering. Answer those four questions and you'll pick the right template without hesitating.

The recursive three-liner handles the whole plain-binary-tree family. Add the iterative BST walk, the two-pointer parent trick, and binary lifting, and you're covered from LeetCode easy through competitive programming. Write them once, deliberately, and then re-derive 1650 from the linked-list intersection idea a week later — that's the part that makes it stick.
