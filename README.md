# Jump Game: The O(n) Greedy Solution, Why It Works, and How Jump Game II, III, and IV Change the Rules

Jump Game is a five-line solution that people get wrong in the explanation, not in the code. You can probably write a version that passes LeetCode 55 right now. The harder part happens when the interviewer asks "why are you allowed to throw away all the other paths?" — and that's exactly where most candidates stall.

This is a walkthrough of the greedy idea behind Jump Game, the boundary conditions that break it, and how the later entries in the series (Jump Game II through VI) reuse or deliberately break the same reasoning.

## What Jump Game actually asks

You get an array `nums`. You start at index 0. Each element is the **maximum** jump length from that position — not a fixed destination. Return `true` if you can reach the last index.


nums = [2, 3, 1, 1, 4]   -> true   (0 -> 1 -> 4)
nums = [3, 2, 1, 0, 4]   -> false  (you land on index 3 sooner or later, and nums[3] == 0)


Constraints are `1 <= nums.length <= 10^4` and `0 <= nums[i] <= 10^5`. That array size is the tell. It's small enough that an O(n²) dynamic programming solution passes, and small enough that a brute-force DFS will not. Interviewers who ask this problem are usually aiming at the O(n) version.

## Why the DP answer is a trap

The natural first solution is a reachability table. `dp[i]` = "can I get to index i?"


dp[0] = true
dp[i] = exists j < i such that dp[j] is true and j + nums[j] >= i


That's O(n²) time and O(n) space. It's correct, and it's the answer a lot of candidates stop at. But look at the inner loop for a second: to decide `dp[i]`, the only thing you need to know about the earlier positions is whether *any* reachable index can land on `i`. You never need to know which one.

That reframing is the whole problem. Because from index `j` you can reach any index in `[j+1, j+nums[j]]`, the set of reachable indices is always a contiguous interval starting at 0 — it can never be a scattered set with holes. So instead of a table, you can carry a single number: the right edge of that interval. Call it `max_reach`.

If `max_reach` is at least `i`, then index `i` is reachable, and every index before it was reachable too. If `i` ever exceeds `max_reach`, the interval has a gap in front of you and nothing later can close it — the answer is `false`.

One variable replaces the entire inner loop. That's O(n) time and O(1) space.

## The forward greedy, line by line

python
def canJump(nums):
    max_reach = 0
    for i, jump in enumerate(nums):
        if i > max_reach:          # gap: index i is unreachable
            return False
        max_reach = max(max_reach, i + jump)
        if max_reach >= len(nums) - 1:
            return True
    return True


Three details worth reading twice, because each one is a real bug in real submissions:

**The failure test is `i > max_reach`, not `i >= max_reach`.** When `max_reach == i`, you've just barely arrived at the current index — that's reachable. Using `>=` falsely fails the single-element input `[0]`, where `i == 0` and `max_reach == 0` and the answer is trivially `true`.

**A zero is not automatically a failure.** `[2, 0, 0]` is `true`: jump two steps from index 0 straight to the end. A zero only kills you if it's the far edge of your frontier and nothing behind you can jump past it. The failing input `[3, 2, 1, 0, 4]` stalls that way — `max_reach` gets stuck at 3, index 4 needs `max_reach >= 4`, and you return `false`.

**The early return is an optimization, not a correctness requirement.** Delete the `max_reach >= len(nums) - 1` line and the loop still returns the right answer; it just keeps walking. Keep it if you like constant factors on inputs like `[10000, ...]`.

## The reverse greedy — same complexity, easier to defend out loud

There's a second O(n) formulation that runs backward from the end. Track a `goal` index, initially the last one. Walk from right to left. If index `i` can reach the current goal (`i + nums[i] >= goal`), then `i` becomes the new goal. At the end, ask whether the goal made it back to 0.

python
def canJump(nums):
    goal = len(nums) - 1
    for i in range(len(nums) - 2, -1, -1):
        if i + nums[i] >= goal:
            goal = i
    return goal == 0


Both versions are O(n) time, O(1) space. The reverse one has a nice property for interviews: it reads like an induction ("if I can reach a reachable index, I'm reachable"), which is often easier to narrate than maintaining a frontier. Pick whichever you can explain without hesitating, and mention the other one if asked for an alternative.

## The part that actually decides the interview

Code you can memorize. The reasoning is what gets graded, and the standard argument is short enough to memorize too:

> Suppose some other strategy reaches an index that the greedy strategy misses. Up to that point, greedy's `max_reach` was at least as large as any alternative's, because greedy always took the maximum over the same set of values. Reachability is monotone: anything reachable from a smaller frontier is reachable from a larger one. So the alternative's entire reachable set is a subset of greedy's, and the supposedly missed index isn't missed at all.

Notice what the argument leans on: the reachable set being an interval, and therefore monotone. Break either assumption and greedy stops being safe. Jump Game III breaks it immediately — you can jump to `i + arr[i]` **or** `i - arr[i]`, so positions behind you can reopen, reachability is no longer a clean prefix, and you need DFS or BFS with a visited marker. Jump Game IV breaks it harder still, letting you teleport to any index holding the same value.

That's the real lesson of the series: the pattern isn't "jump problems are greedy." The pattern is "reachability with a monotone frontier is greedy; reachability without one is a graph search."

## The Jump Game family at a glance

| # | Problem | What changes | Pattern | Time / Space |
| --- | --- | --- | --- | --- |
| 55 | Jump Game | Reach the end, yes/no | Forward max-reach greedy, or reverse greedy | O(n) / O(1) |
| 45 | Jump Game II | Minimum number of jumps | BFS-style greedy over level boundaries | O(n) / O(1) |
| 1306 | Jump Game III | Jump `i ± arr[i]`, reach any index valued 0 | DFS/BFS with visited marking | O(n) / O(n) |
| 1345 | Jump Game IV | Move `i ± 1`, or jump to any equal value | BFS + value-to-indices hash map, with bucket pruning | O(n) / O(n) |
| 1340 | Jump Game V | Any start, land only on smaller values, distance limited | DFS + memoization | O(n·d) / O(n) |
| 1696 | Jump Game VI | Maximize score, jump at most `k` forward | DP + monotonic deque | O(n) / O(n) |

Two notes on the entries people fumble most.

**Jump Game IV (1345)** is the one that punishes a naive BFS. Indexing by value gives you a bucket of every index holding that value, and a plain BFS then re-explores the same bucket from every element in it — quadratic on inputs like eleven identical sevens. The fix is to clear the bucket once you've processed any index in it, because every index sharing that value has the same distance. That single line takes the worst case from O(n²) to O(n).

**Jump Game VI (1696)** looks like a sliding-window maximum, and it basically is one. `dp[i] = nums[i] + max(dp[i-k..i-1])`, computed with a deque that holds indices in decreasing `dp` order. Drop indices that fell out of the `k` window, and pop from the back while the back's value is `<=` the current one — `<=`, not `<`, since a newer index with an equal value stays valid longer.

## Jump Game II: counting jumps instead of checking reachability

Jump Game II asks for the minimum number of jumps, and the answer is what the greedy for 45 would be if you counted in levels. One frontier isn't enough: you also need to know where the current jump ends.

python
def jump(nums):
    if len(nums) <= 1:
        return 0
    jumps = 0
    current_end = 0    # boundary of the jump you're currently committed to
    farthest = 0       # farthest index any position up to current_end can reach
    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])
        if i == current_end:
            jumps += 1
            current_end = farthest
            if current_end >= len(nums) - 1:
                break
    return jumps


This is breadth-first search with the queue removed. Positions reachable in 0 jumps are level 0, positions reachable in 1 jump are level 1, and so on — and because indices get visited in order, `current_end` is the end of the current level while `farthest` is the end of the next one. You only increment the counter when you're forced to cross a boundary. Same O(n) time, O(1) space as the base problem, one extra variable.

If you want the version with an editable template and the level-boundary reasoning spelled out, 👉 [read AlgoMonster's full Jump Game walkthrough](https://algo.monster/liteproblems/55?fpr=li69) — the problem page is public, and their Jump Game VI page covers the deque variant if you want to compare the two.

## Spotting this pattern next time

Ask three questions when you see a new problem in this shape:

1. **Is the question "can you reach" or "minimum steps to reach"?** Reachability with a single moving frontier is a greedy scan. Minimum steps is a level-by-level greedy or a BFS.
2. **Is the reachable set an interval?** If moves only go forward in a bounded range, yes, and a single `max_reach` variable is enough. If moves can go backward or teleport, no — reach for DFS/BFS.
3. **Does the score depend on which position you came from?** Jump Game VI does, which is why it needs DP plus a data structure to query the best predecessor inside a window.

Getting fast at question 2 is the difference between recognizing this family in the interview and writing O(n²) DP under a clock.

## Where AlgoMonster fits in

AlgoMonster is a text-and-diagram interview prep platform from engineers who worked at Google, built around named patterns rather than an endless problem queue. Jump Game and Jump Game II sit inside its **Greedy** pattern module, which is the point — the platform teaches the interval-monotonicity argument above as a reusable template, so the same reasoning covers Gas Station and the rest of the family instead of being memorized per problem.

What you actually get on the paid tiers: all patterns, lessons, and problems unlocked, unlimited AI assistant usage, unlimited Speedrun (timed recall drills), company question banks that get refreshed monthly, and roughly 3,000 editorials with several hundred illustrated walkthroughs. Two bonus courses ship with a subscription — a CS fundamentals track and a "build an LLM from scratch" course.

Here's the current lineup, pulled from the official subscription page:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited subset of patterns, lessons, and problems — enough to test the teaching style | $0 | No commitment | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks; 3000+ editorials; 3-day free trial first | $99/year (~$8.25/mo) | Billed annually | [Start the Pro Yearly trial](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in Pro Yearly, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 | One-time payment | [Get Pro Lifetime access](https://algo.monster/subscribe?fpr=li69) |

Two honest caveats. First, pricing on this site moves with seasonal promos — third-party deal trackers list the list prices as $198/year and $378 lifetime with the current sale cutting that roughly in half, so confirm the number at checkout rather than trusting any blog, including this one. Second, the "referral" on the Lifetime plan is worded as being for qualified candidates on the official page; some reviews describe it as a broader job guarantee, which is a stronger claim than the site makes.

Who benefits: readers who'd rather work through a written template and diagram than a 40-minute video, and people prepping on a 4–8 week clock who need a finite curriculum. Who doesn't: video learners (there's no video content at all), and anyone prepping for senior system design rounds — that coverage is introductory here, not a substitute for a dedicated resource.

👉 [Compare the plans on AlgoMonster before you commit](https://algo.monster/subscribe?fpr=li69)

## Quick answers

**Is greedy always correct for Jump Game?** For the original problem, yes — the exchange argument holds because the reachable set is a contiguous interval. It stops holding the moment the problem allows backward moves or value-based teleports, which is where Jump Game III and IV send you to graph search.

**Why does my solution fail on `[0]`?** The failure check is almost always `i >= max_reach` instead of `i > max_reach`, or the loop is structured to return `false` before it ever considers that the goal equals the start.

**Do I need two variables for Jump Game II?** Yes, one for the end of the current level and one for the farthest index the next level can reach. A single variable can't tell you when a jump is mandatory, only how far you could go if you took one.

The code for Jump Game is short enough to write from memory. The explanation is what separates a pass from a follow-up question you can't answer — so learn the interval argument, not just the five lines.
