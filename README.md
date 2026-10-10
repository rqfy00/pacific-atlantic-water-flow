# Pacific Atlantic Water Flow: Why Brute Force TLEs, and the Two Reverse Traversals That Solve It in O(m·n) With DFS or BFS

Pacific Atlantic Water Flow looks like a small problem. A grid of numbers, two oceans, one question: which cells drain into both? Then you write the obvious solution, submit, and watch a wall of red.

The reason usually isn't that you picked the wrong algorithm. It's that you ran it in the wrong direction. Water flows downhill, but the efficient solution walks uphill from the ocean, and that single inversion is the whole trick.

## What the problem is actually asking

LeetCode 417 gives you an `m x n` integer matrix `heights`, where `heights[r][c]` is the elevation above sea level of one cell.

- The Pacific Ocean touches the island's **top and left edges**.
- The Atlantic Ocean touches the island's **bottom and right edges**.
- Rainwater can move north, south, east, or west to a neighbouring cell, but only if that neighbour's height is **less than or equal to** the current cell's height. Flat ground is fine; uphill is not.
- Any cell sitting on the island's edge can dump water straight into whichever ocean it borders.

You need to return the coordinates of every cell from which water can reach *both* oceans. Order doesn't matter.

The canonical example:


heights = [
  [1,2,2,3,5],
  [3,2,3,4,4],
  [2,4,5,3,1],
  [6,7,1,4,5],
  [5,1,1,2,4]
]


The answer is `[[0,4],[1,3],[1,4],[2,2],[3,0],[3,1],[4,0]]`. Take `[2,2]` (height 5, the peak in the middle): it can drop to `[1,2]` and then `[0,2]`, which is on the top edge and therefore touches the Pacific. It can also drop to `[2,3]` and `[2,4]`, which sits on the right edge and touches the Atlantic. Two separate downhill paths, one cell.

Constraints are `1 <= m, n <= 200` with `0 <= heights[r][c] <= 10^5`. Hold onto that 200 figure, because it's what turns the naive approach into a timeout.

## Why checking every cell doesn't work

The straightforward idea is to do exactly what the problem describes. For each cell, run a search downhill, and record whether you ever step off the top/left boundary and whether you ever step off the bottom/right boundary. If both happen, keep the cell.

It's correct. It's also unusable.

Every one of those searches can potentially touch every cell in the grid, so a single traversal costs O(m·n). You're running it from every one of the m·n starting cells, which gives you O((m·n)²). On the maximum 200×200 grid that's 40,000 cells squared — 1.6 billion visit operations in the worst case. LeetCode will not wait for you.

Stack overflow is a second, quieter failure mode. A cell in the middle of a plateau of equal heights can walk through the entire grid before it terminates, and a recursive DFS doing 40,000 levels deep in Python blows past the default recursion limit of 1,000 long before you ever see a "wrong answer".

## The flip: let the oceans do the walking

Here's the observation that fixes everything.

Reachability here is symmetric in a useful way. If water can trickle from cell X all the way down to the Pacific edge, then walking outwards from the Pacific edge — but only onto cells that are **at least as high** as the one you're standing on — will eventually arrive at X. Climbing uphill in reverse is the same relation as flowing downhill forwards, just read the other way.

So instead of asking 40,000 separate questions, you ask two:

1. Starting from every cell on the top row and left column, flood inward with the rule `heights[neighbour] >= heights[current]`. Everything you mark drains to the Pacific.
2. Do the same from the bottom row and right column. Everything you mark drains to the Atlantic.

The answer is the intersection of those two sets. Each cell gets visited at most once per ocean, so the total work is O(m·n) — linear in the size of the grid, which is the complexity LeetCode expects.

Two details that trip people up here. First, the inequality genuinely flips: forward flow allows `next <= current`, so reverse flow allows `next >= current`. Second, the corner cells belong to both oceans — `[0][0]` borders the Pacific on both its top and left sides, `[m-1][n-1]` borders the Atlantic on both sides, and the other two corners each touch one ocean twice. Seeding only the four corners is a common shortcut and it produces wrong answers.

## The DFS version in Python

python
class Solution:
    def pacificAtlantic(self, heights: List[List[int]]) -> List[List[int]]:
        if not heights:
            return []
        ROWS, COLS = len(heights), len(heights[0])
        pac, atl = set(), set()

        def dfs(r, c, visited):
            visited.add((r, c))
            for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                nr, nc = r + dr, c + dc
                if (0 <= nr < ROWS and 0 <= nc < COLS
                        and (nr, nc) not in visited
                        and heights[nr][nc] >= heights[r][c]):
                    dfs(nr, nc, visited)

        for c in range(COLS):
            dfs(0, c, pac)          # top edge -> Pacific
            dfs(ROWS - 1, c, atl)   # bottom edge -> Atlantic

        for r in range(ROWS):
            dfs(r, 0, pac)          # left edge -> Pacific
            dfs(r, COLS - 1, atl)   # right edge -> Atlantic

        return [[r, c] for r in range(ROWS) for c in range(COLS)
                if (r, c) in pac and (r, c) in atl]


Reading it top to bottom: the `visited` set doubles as both the "have I been here" guard and the output of the traversal, which is why there's no separate boolean grid. Each `dfs` call marks its own cell immediately and then recurses into neighbours that are unvisited and no lower than the current cell.

The four loops seed the borders. Two columns of seeds feed the Pacific (`0, c` and `r, 0`), two feed the Atlantic (`ROWS-1, c` and `r, COLS-1`). Corners get seeded twice, which is harmless because the `not in visited` check stops the second call immediately.

If you prefer an explicit boolean grid over a set, that's fine too. A `[[False]*COLS for _ in range(ROWS)]` per ocean does the same job with slightly better constant factors and worse readability, and it's the version you'll see in most published editorials.

### Tracing the example

Run the Pacific flood on the 5×5 grid above. The top row seeds `[0,0]` through `[0,4]`, the left column seeds `[1,0]` through `[4,0]`. Climbers move only onto equal-or-higher ground, so from `[0,0]` (height 1) you can climb to `[1,0]` (3) and `[0,1]` (2), and from there the flood spreads through most of the left and middle of the grid.

The Atlantic flood seeds the bottom row and right column, climbing leftwards and upwards from heights 4, 5, 1, 1, 2, 4 and the right-hand column.

Overlay the two and you get `[0,4]`, `[1,3]`, `[1,4]`, `[2,2]`, `[3,0]`, `[3,1]`, `[4,0]`. The two cells at `[0,0]` and `[4,4]` are a good sanity check: they're corners, so each is trivially reachable from one ocean but has to earn its way into the answer through the other.

## BFS does the same thing with a queue

Multi-source BFS is the same algorithm with the recursion replaced by a deque, and instead of seeding one cell at a time you push all the border cells in first.

python
from collections import deque

class Solution:
    def pacificAtlantic(self, heights: List[List[int]]) -> List[List[int]]:
        ROWS, COLS = len(heights), len(heights[0])

        def bfs(sources):
            reachable = set(sources)
            q = deque(sources)
            while q:
                r, c = q.popleft()
                for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                    nr, nc = r + dr, c + dc
                    if (0 <= nr < ROWS and 0 <= nc < COLS
                            and (nr, nc) not in reachable
                            and heights[nr][nc] >= heights[r][c]):
                        reachable.add((nr, nc))
                        q.append((nr, nc))
            return reachable

        pac_src = [(0, c) for c in range(COLS)] + [(r, 0) for r in range(ROWS)]
        atl_src = [(ROWS - 1, c) for c in range(COLS)] + [(r, COLS - 1) for r in range(ROWS)]

        pac, atl = bfs(pac_src), bfs(atl_src)
        return [[r, c] for r in range(ROWS) for c in range(COLS)
                if (r, c) in pac and (r, c) in atl]


Same complexity, same output. The practical difference matters more than the theory in Python.

|  | Recursive DFS | Iterative BFS |
| --- | --- | --- |
| Worst-case stack depth | Up to m·n frames — a 200×200 plateau can mean 40,000 recursive calls and a `RecursionError` | None; the queue lives on the heap |
| Extra setup needed | Sometimes `sys.setrecursionlimit(10000)`, which trades a clean error for a potential C-stack crash | Nothing |
| Interview impression | Shows you're comfortable with recursion | Shows you know when not to use it |
| Code length | Shorter | A few lines longer |

If you're writing Python under interview pressure on a grid that can be 40,000 cells, the queue version removes a whole class of failure for free. Keep the DFS version in your head as the explanation and the BFS version as the implementation.

## Complexity, and why O(m·n) isn't a typo

| Approach | Time | Space | Verdict |
| --- | --- | --- | --- |
| Search downhill from every cell | O((m·n)²) | O(m·n) per traversal | Correct, times out at 200×200 |
| Reverse DFS from both oceans | O(m·n) | O(m·n) for the two visited sets, plus stack | Accepted |
| Reverse BFS from both oceans | O(m·n) | O(m·n) | Accepted, no recursion risk |

The linear bound comes from the visited set doing real work. In the DFS version, a cell that's already marked returns immediately on any later call, so no cell is expanded twice within one ocean's flood. Two floods means each cell is expanded at most twice, and `2·m·n` is still O(m·n).

Space is dominated by the two visited sets. If you use Python sets of tuples, the constant factor is worse than two boolean grids — around 40,000 `(r, c)` tuples per ocean at the maximum grid size — but both are O(m·n) and both pass comfortably.

## The bugs that actually cost people this problem

**Comparing in the wrong direction.** This is the number one cause of a solution that compiles, runs, and returns nonsense. Forward flow allows `next <= current`. Reverse flow allows `next >= current`. If you copy the comparison straight from the problem statement into a border-seeded traversal, you'll mark the wrong cells and probably get something that looks plausible enough to waste ten minutes debugging.

**Seeding only the four corners.** Corner cells are a subset of the border, not the whole thing. A cell halfway along the top row touches the Pacific directly, and if you never seed it, nothing discovers it.

**Mixing up which edges belong to which ocean.** Pacific = top + left. Atlantic = bottom + right. It's easy to write `(r, 0)` into the Atlantic set without noticing, because the two loops look nearly identical.

**Using one shared visited set.** If both floods write into the same set, the second flood can't tell which cells the first one already claimed, and your intersection is meaningless. Two sets, one per ocean.

**Forgetting the single-cell and single-row cases.** With `heights = [[1]]`, the only cell borders both oceans, so the answer is `[[0,0]]`. With `heights = [[1],[1]]`, it's `[[0,0],[1,0]]`. Both fall out of the general code for free — but only if your border loops handle `m == 1` or `n == 1` without indexing off the end.

## Recognizing this pattern before you start typing

The tell for multi-source reverse traversal is a question shaped like *"find every node that can reach a target set."* Whenever the target is a whole boundary rather than one specific node, run the search from the boundary instead of towards it. Grid questions that ask about cells touching an edge, cells flooded from multiple sources, or cells that survive a spreading process are the same shape.

You'll see the same move in `Rotting Oranges` and `01 Matrix` (multiple sources entering the queue at once), and in `Surrounded Regions`, where the border-seeded flood is what saves you from rewriting the board repeatedly. Learn the pattern once and each of those becomes a variation on a template rather than a fresh puzzle.

That framing is the entire pitch behind [👉 AlgoMonster's walkthrough of Pacific Atlantic Water Flow](https://algo.monster/liteproblems/pacific-atlantic-water-flow?fpr=li69), which lays out the reverse-flow intuition with diagrams before it drops the code on you. The platform was built by ex-Google engineers specifically around patterns instead of volume: 48 explicit patterns, roughly 231 lessons and problems, and around 700 illustrations and flowcharts. There are no videos anywhere on the site — everything is text and diagrams, which is either exactly what you want or a dealbreaker, depending on how you learn.

The other thing worth knowing before you consider paying: reviews consistently flag three gaps. The problem bank is small next to LeetCode's thousands, the system design content is introductory rather than enough for senior loops, and there's no timed mock-interview environment. Several review sites also mention a job guarantee, though the terms are worth reading yourself rather than trusting a summary.

## AlgoMonster plans and pricing

Plans and prices below are what's shown on the subscription page when you arrive through the link in this article. AlgoMonster runs near-constant promotions and the numbers move, so the checkout page is the only figure that actually counts.

| Plan | What you unlock | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge whether the text-and-diagram format clicks for you | $0 | No card needed to look around | [Try the free tier first](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question sets refreshed monthly; 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials; member discount on live bootcamps; 12 months of full access | $5.84/mo, billed annually ($69.99/year) | Annual, renews | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, paid once | $89 | One-time, yours permanently | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

A caveat you should factor in: third-party reviews published over the same period quote wildly different numbers, from roughly $89 to $299 a year and $99 to $459 for lifetime. Those reviews are describing sale prices at the moment they were written. AlgoMonster doesn't offer monthly billing at all, so the real choice is annual versus one-time.

Which one makes sense depends on your timeline. If you have an interview in six weeks and you'll be done with the material afterwards, the annual plan covers it and costs less upfront. If you're the kind of engineer who re-preps every couple of years when you start looking at the market, the lifetime option pays for itself after roughly a year and a half of renewals at the annual rate. There's no team or enterprise tier — this is a product for individuals.

## Questions people ask once they've seen the solution

**Why does my DFS time out even though it's "just a grid"?** Because you're running a full traversal per cell. Check that you're seeding only the borders and that each flood shares one visited set across all its seed cells. If your visited grid is being recreated inside the per-cell loop, you've accidentally written the O((m·n)²) version.

**Do I need two visited structures?** Yes. A cell reachable from the Pacific and a cell reachable from the Atlantic are different facts, and you need to compare them at the end. One set collapses that distinction.

**Can I do it in a single traversal?** Not cleanly. You'd need to propagate two different ocean flags simultaneously and handle cells that gain the second flag after the first pass has already moved past them, which pushes you back toward repeated work. Two floods and an intersection is simpler and just as fast.

**Does the order of the returned coordinates matter?** No, and LeetCode accepts any order. That's why both solutions above iterate the grid row by row and append as they find cells.

**What if every cell is the same height?** Then every cell can reach every edge, so the entire grid is the answer. It's also the case that gives a recursive DFS its maximum stack depth, which is exactly why the BFS version is the safer submission.

## Bottom line

Pacific Atlantic Water Flow is a reachability question wearing a grid costume, and the only thing separating the slow solution from the fast one is the direction you traverse. Seed the borders, climb uphill, intersect the two sets, and you're at O(m·n) with about twenty lines of Python.

If you can already write that from memory, you don't need a course for this problem. If the *reason* the reversal works still feels like something you memorized rather than understood, that's the gap worth closing — and it's the one that shows up again in `Surrounded Regions`, `01 Matrix`, and every other border-seeded flood you'll meet later. [👉 See how AlgoMonster diagrams the reverse-flow idea](https://algo.monster?fpr=li69) and check the free tier against your own prep timeline before spending anything.
