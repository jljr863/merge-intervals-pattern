# merge intervals python: the sort-and-scan solution, the edge cases that break it, and the pattern behind LeetCode 56

Most people land on this problem one of two ways. Either they're on LeetCode 56 trying to write something that doesn't fail on `[[1,4],[4,5]]`, or they've written a merge function for a scheduling feature at work and need to know why overlapping bookings are coming out wrong.

Both cases want the same thing: the actual algorithm, the Python gotchas that only show up when you run it, and enough context to recognize the same shape again next time. Here's all of that.

## What "merge intervals" is actually asking

You get `intervals[i] = [start_i, end_i]`. You return a list of non-overlapping intervals that cover everything the input covered.


Input:  [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]


`[1,3]` and `[2,6]` overlap, so they collapse into `[1,6]`. The other two don't touch anything, so they pass through unchanged. On LeetCode the constraints are small — up to 10,000 intervals, each with two values, starts and ends between 0 and 10,000 — which means an O(n log n) solution isn't just acceptable, it's the expected answer.

One ambiguity worth deciding before you write code: **do touching intervals count as overlapping?** In LeetCode 56, yes. `[[1,4],[4,5]]` becomes `[[1,5]]`, because the problem treats closed intervals and a shared endpoint as a merge. In a real scheduling system, `[9:00–10:00]` and `[10:00–11:00]` usually means back-to-back, not a conflict. Same code shape, different comparison operator. If it's an interview, ask.

## The Python solution, and why each line is there

python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    if not intervals:
        return []

    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0][:]]

    for start, end in intervals[1:]:
        last = merged[-1]
        if start <= last[1]:
            last[1] = max(last[1], end)
        else:
            merged.append([start, end])

    return merged


The empty check comes first because the next line indexes `intervals[0]`. Without it you get an `IndexError` on the `[]` case, which is the test case everyone forgets.

`intervals.sort(key=lambda x: x[0])` sorts in place by start value. Sorting is the entire idea, and the next section covers why.

`merged = [intervals[0][:]]` seeds the result with a **copy** of the first interval. Drop the `[:]` and you're storing a reference to `intervals[0]`, so when you later do `last[1] = max(...)`, you're mutating the caller's input list. It still returns the right answer for the tests, which is what makes it such an annoying bug — your function quietly rewrites the array it was handed.

Inside the loop, `start <= last[1]` is the overlap test, and it uses `<=`, not `<`. For LeetCode 56 that equality is the difference between passing and failing `[[1,4],[4,5]]`.

`last[1] = max(last[1], end)` is doing more work than it looks like. You can't just assign `end`, because an interval can be fully swallowed by the one you're merging into:


merged = [[1,10]], next = [2,3]
start <= last[1]  →  2 <= 10, merge
last[1] = max(10, 3) = 10   →  [[1,10]]


Assign `end` directly and you get `[[1,3]]`, which has thrown away everything between 3 and 10.

If you don't want to touch the input at all, this version avoids both the sort-in-place and the aliasing question:

python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    result = []
    for start, end in sorted(intervals):
        if result and start <= result[-1][1]:
            result[-1][1] = max(result[-1][1], end)
        else:
            result.append([start, end])
    return result


`sorted(intervals)` with no key works here — Python compares lists element by element, so it sorts by start first and by end as a tiebreaker. The tiebreaker never changes the output for this problem, which is why you'll see both forms in the wild.

## Why sorting first is the whole trick

After sorting by start, one invariant holds: every interval still in `merged` has a start value less than or equal to the start of anything you haven't processed yet. That means you only ever need to look at the **last** interval in your result. No searching backward, no keeping track of a range of pending intervals, no second pass.

The overlap check `start <= last[1]` is only sufficient because of that sort. Un-sorted input has no such guarantee, and this exact code will happily merge `[8,10]` into `[1,6]` if `[8,10]` shows up one position too early in the array.

## Edge cases worth testing before you submit

| Input | Expected | What it's testing |
| --- | --- | --- |
| `[]` | `[]` | Empty guard, indexing safety |
| `[[1,3]]` | `[[1,3]]` | Single interval, loop body never runs |
| `[[1,4],[4,5]]` | `[[1,5]]` | The `<=` vs `<` decision |
| `[[1,10],[2,3]]` | `[[1,10]]` | Nested interval, needs `max` |
| `[[1,4],[0,4]]` | `[[0,4]]` | Second start comes before the first |
| `[[1,2],[3,4]]` | `[[1,2],[3,4]]` | No overlap, nothing merges |
| `[[3,5],[1,2],[6,8],[2,4]]` | `[[1,5],[6,8]]` | Unsorted input |
| `[[1,4],[1,4]]` | `[[1,4]]` | Exact duplicates |

The nested case and the touching case are where most wrong submissions die. Everything else tends to fall out of the sort.

## Complexity

Time is **O(n log n)**, and all of it goes to the sort. The merge loop is a single O(n) pass. Space is **O(n)** for the output — and note that in Python, `list.sort()` also allocates a temporary buffer in the worst case, so a "sort in place" implementation isn't quite the O(1) extra space it is in C++.

If the problem guarantees sorted input, that's a different conversation. LeetCode 57 (`Insert Interval`) does exactly that, and the standard O(n) answer there is a three-phase walk: copy intervals that end before the new one starts, merge everything that overlaps, copy the rest. No sort needed.

## Six variations of the same skeleton

Once you can spot the sort-then-scan shape, a lot of "different" interval problems collapse into it:

| Problem | LeetCode | What changes |
| --- | --- | --- |
| Merge Intervals | 56 | Sort by start, merge the last interval in the result |
| Insert Interval | 57 | Input already sorted; three-phase O(n) walk |
| Non-overlapping Intervals | 435 | Sort by **end**, greedily keep the earliest finisher |
| Meeting Rooms II | 253 | Sort starts and ends into separate arrays, sweep with two pointers |
| Interval List Intersections | 986 | Two pointers across two sorted lists |
| Minimum Number of Arrows to Burst Balloons | 452 | Sort by end, count arrows |

The trap that catches people across all of these: sorting by the wrong key. Sort by **start** for merging and intersection, by **end** for anything that says "remove the fewest" or "select the most non-overlapping." Picking the wrong one produces code that looks fine and fails on three test cases.

## AlgoMonster is built around this exact shape

The reason `merge intervals` shows up in so many interview prep discussions is that it's the archetype for an entire category. AlgoMonster treats it that way — Merge Intervals and Insert Interval sit inside an Interval pattern section with its own writeup, step-by-step reasoning, and flowchart, alongside discussion threads where people compare O(n) and O(n log n) approaches.

That's the actual pitch of the site: instead of grinding problems one at a time, you learn the pattern, then apply the template to every problem wearing the same costume. The explanations are text and diagram based rather than video, across six courses, and the editorial library runs into the thousands. If you learn by reading and want the code templates in your own language, it's a reasonable fit; if you need someone talking you through a whiteboard, it isn't.

Here's what's currently on the subscribe page:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge whether the text-and-diagram format works for you | $0 | — | [Start with the free content](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; real company questions updated monthly; lessons across six courses; member discount on live bootcamps | $99/year (about $8.25/mo billed annually) | 12 months, with a 3-day free trial on the yearly plan | [Check the yearly plan price](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 one-time | Billed once | [See the lifetime option](https://algo.monster?fpr=li69) |

> Two caveats worth knowing before you budget for this. AlgoMonster runs rotating sales, and the subscribe page has displayed lower prices than the ones above during promotions — a snapshot of the same page showed roughly $69.99/year and $89 for lifetime. Partner sites also publish discount codes that stack on top of the current sale, but those are third-party claims, so the number that matters is whatever your checkout screen shows. Prices and included content change; check the live page.

There's no monthly plan and no team tier. It's annual or lifetime, which makes the decision fairly simple: if your interviews are over in the next few months, yearly. If you plan to switch jobs repeatedly over the next few years, the lifetime price is roughly two years of the annual one, so the crossover isn't far off.

## Common mistakes in Python specifically

A short list of the things that actually break people, as opposed to generic advice:

`intervals = intervals.sort(key=lambda x: x[0])` — `list.sort()` returns `None`. Your intervals are now sorted and your variable is now empty, and the error message points somewhere useless.

`merged = [intervals[0]]` — mutation of the caller's data, as covered above. Works fine on LeetCode, surprising in production.

`for a, b in intervals: a[1] = ...` — if any interval is a tuple, this raises `TypeError: 'tuple' object does not support item assignment`. Tuples are common if the data came out of a database or a JSON config.

`merged.append([start, end])` before the overlap check — this produces one output entry per input interval and hides the fact that nothing ever merges.

`last[1] = end` instead of `max(last[1], end)` — passes the sample cases, fails on nesting.

And a broader one: Python's standard library has no interval-merging function. The `interval` package on PyPI exists, and pandas has interval indexing if you're already working with dataframes, but for interview purposes you're writing the loop yourself.

## How to actually practice this one

Read the code until you can write it from memory, then stop looking and reproduce it. Test it against the eight edge cases above before you submit anything. Then, once it passes, do the exercise that actually builds pattern recognition: solve `Insert Interval`, `Non-overlapping Intervals` and `Meeting Rooms II` on the same day, and note out loud where the sort key changes and why. Three problems in a row that share one skeleton will teach you more than twenty random ones.

If you want the pattern explanations and the flowchart that maps problem wording to approach, that's what the 👉 [AlgoMonster interval pattern material](https://algo.monster?fpr=li69) covers. If you'd rather just keep writing the twenty random problems, the code above will still pass LeetCode 56 — you'll just be re-deriving it each time.
