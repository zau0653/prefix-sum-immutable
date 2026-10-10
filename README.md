# range sum query immutable: prefix sum template, O(1) queries, and the off-by-one that fails your tests

Most people land on this problem the same way: LeetCode 303, "Range Sum Query - Immutable," and a nagging feeling that the obvious solution is missing something. It is. The brute force answer passes small tests and dies on the real ones, and the fix is one of the few interview patterns you'll reuse in half a dozen other problems.

Here's the whole idea, plus the parts most write-ups skip: why the prefix array has an extra slot, where the off-by-one actually bites, and which follow-up problems punish you if you only memorized the formula.

## What LeetCode 303 actually asks you to build

You're given an integer array `nums` and asked to implement a class with two methods:

- `NumArray(int[] nums)` — constructor, initializes the object with the array
- `int sumRange(int left, int right)` — returns `nums[left] + nums[left + 1] + ... + nums[right]`, inclusive on both ends

The canonical example uses `nums = [-2, 0, 3, -5, 2, -1]` [1]:

| Call | Reasoning | Result |
| --- | --- | --- |
| `sumRange(0, 2)` | -2 + 0 + 3 | 1 |
| `sumRange(2, 5)` | 3 + (-5) + 2 + (-1) | -1 |
| `sumRange(0, 5)` | sum of everything | -3 |

The constraints are what make this an interview question rather than a warm-up [2]:

- `1 <= nums.length <= 10^4`
- `-10^5 <= nums[i] <= 10^5`
- `0 <= left <= right < nums.length`
- At most `10^4` calls to `sumRange`

Note the last two lines together. The array never changes — that's the "immutable" in the title — and the queries come in bulk. The problem tags it as Array, Design, and Prefix Sum [3], which is a fairly direct hint about where you're supposed to end up.

## Brute force: correct, and still not good enough

The straightforward solution stores the array and loops on every query:

python
class NumArray:
    def __init__(self, nums):
        self.nums = nums

    def sumRange(self, left, right):
        res = 0
        for i in range(left, right + 1):
            res += self.nums[i]
        return res


That's O(n) per query and O(1) extra space [4]. It's also the version that gets you a time limit exceeded once the grader runs its full test suite.

Do the arithmetic. `10^4` elements, up to `10^4` queries, worst case each query spanning nearly the whole array. That's on the order of `10^8` additions, and each one is a separate Python-level operation. The repeated work is the problem: every query re-adds elements it already summed five times before.

## The prefix sum fix: pay once, query forever

Precompute running totals in the constructor, then answer any range with a single subtraction.

Let `prefix[k]` be the sum of the first `k` elements — that is, `nums[0] + nums[1] + ... + nums[k-1]`. Building it is one pass:


prefix[0] = 0
prefix[k] = prefix[k - 1] + nums[k - 1]


Then the sum of `nums[left..right]` is:


prefix[right + 1] - prefix[left]


Why it works: `prefix[right + 1]` is the total of everything up to and including `right`. `prefix[left]` is the total of everything strictly before `left`. Subtract the second from the first and the front you didn't ask for cancels out, leaving exactly the range [5].

With the example array, the prefix array looks like this:

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Value | 0 | -2 | -2 | 1 | -4 | -2 | -3 |

`sumRange(2, 5)` becomes `prefix[6] - prefix[2] = -3 - (-2) = -1`. Same answer as the loop, no iteration.

AlgoMonster's write-up of this problem walks the same arithmetic with a different array, tracking `s = [0, 3, 8, 10, 18, 22]` and verifying `sumRange(1, 3)` as `s[4] - s[1] = 18 - 3 = 15` [6]. Worth reading if you want the formula shown against concrete numbers rather than abstract indices — 👉 [AlgoMonster's Range Sum Query walkthrough](https://algo.monster/liteproblems/303?fpr=li69).

### Why the prefix array has n + 1 slots

This is the detail that separates people who understand prefix sums from people who copied a template.

If you only allocate `n` slots, you have no clean value to represent "the sum of zero elements," and the `left = 0` case has to be special-cased with an `if`. Allocating `n + 1` slots with `prefix[0] = 0` makes the empty prefix a real, addressable value, so the formula `prefix[right + 1] - prefix[left]` works uniformly for every valid input, including queries that start at index 0 [5].

AlgoMonster's prefix sum primer puts the boundary rule plainly: the prefix array is length `n + 1`, `prefix[0]` is 0, and when a problem hands you an inclusive right index `r`, you translate it to `prefix[r + 1] - prefix[i]` [5]. That inclusive-to-exclusive conversion is the single most common place this problem goes wrong.

### Working solutions

Python, using `itertools.accumulate` with an initial zero [3]:

python
from itertools import accumulate

class NumArray:
    def __init__(self, nums):
        self.prefix = list(accumulate(nums, initial=0))

    def sumRange(self, left, right):
        return self.prefix[right + 1] - self.prefix[left]


Java:

java
class NumArray {
    private int[] prefix;

    public NumArray(int[] nums) {
        prefix = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }

    public int sumRange(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
}


C++, same shape:

cpp
class NumArray {
    vector<int> prefix;
public:
    NumArray(vector<int>& nums) {
        prefix.resize(nums.size() + 1, 0);
        for (int i = 0; i < nums.size(); i++)
            prefix[i + 1] = prefix[i] + nums[i];
    }
    int sumRange(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
};


One heads-up on AlgoMonster specifically: a thread in their own discussion forum flags that the Java snippet shown on their Range Sum Query problem page builds and returns the prefix array rather than returning the range answer, which doesn't match the method signature the problem asks for [7]. The explanation on that page is fine — the snippet is what's off. If you're copying code from any platform, including that one, check what the method is supposed to return before you paste.

## The off-by-one traps, in the order they'll hit you

Three mistakes account for nearly every failed submission on this problem.

1. **Confusing inclusive and exclusive right bounds.** The problem gives you an inclusive `right`. The formula wants `prefix[right + 1]`. Forget the `+1` and you're off by exactly one element, which usually still passes the sample and fails everything else.
2. **Special-casing `left == 0` instead of allocating the extra slot.** If your solution contains `if left > 0` inside `sumRange`, you allocated `n` cells instead of `n + 1`. It works, but it means you're carrying a branch through every query for no reason.
3. **Rebuilding the prefix array per query.** The precompute belongs in the constructor. Moving it inside `sumRange` turns an O(n) setup into O(n) per call and quietly restores the original problem.

## Complexity, and when prefix sums are worth the setup

| Approach | Build | Per query | Extra space |
| --- | --- | --- | --- |
| Brute force | O(1) | O(n) | O(1) |
| Prefix sum | O(n) | O(1) | O(n) |

For `n = 10^4` and `q = 10^4`, brute force is roughly O(n·q) ≈ 10^8 operations; prefix sum is O(n + q) ≈ 2·10^4 [4]. The trade is one pass and one array of extra memory.

But here's the honest caveat: for a single one-off range sum, the plain loop is simpler and you should just use it. The prefix array earns its O(n) setup only when the same array gets queried repeatedly, or when an algorithm needs to search across subarray boundaries [5]. Choosing prefix sums when you have one query is the same category of mistake as choosing brute force when you have ten thousand.

## Follow-ups that reuse this exact pattern

Prefix sum isn't a single problem, it's a family. Four you'll meet next:

- **304. Range Sum Query 2D - Immutable.** The same idea on a grid. Instead of a 1D prefix array you build a 2D one, and the rectangle sum becomes an inclusion–exclusion of four cells. Same "subtract the front you didn't ask for" logic, two dimensions instead of one.
- **307. Range Sum Query - Mutable.** The array changes now, so a static prefix array is useless — you'd rebuild it after every update. This is where segment trees and Fenwick trees (BITs) take over. If 303 clicked, 307 is the natural next step, and the contrast is the lesson: prefix sums are for static data.
- **560. Subarray Sum Equals K.** The running-sum-plus-hash-map upgrade. You record each prefix total as you scan and ask whether a complementary value appeared earlier, converting a subarray search into O(1) lookups [5].
- **Product of Array Except Self.** Prefix products from both directions. Same shape, different operation.

The generalization beyond addition is worth internalizing: prefix XOR and prefix products work identically because they're all associative operations with an identity element. Prefix sums specifically require subtraction to be reversible, which is why prefix min/max doesn't decompose the same way.

## Where AlgoMonster fits into studying this

Range Sum Query - Immutable sits inside AlgoMonster's Prefix Sum module, alongside the primer, the Target Sum subarray variant, and Product of Array Except Self [5]. The platform is built by ex-Google engineers around exactly this kind of pattern grouping rather than a flat problem bank — 48 coding patterns, about 231 lessons and problems, and roughly 670–700 interactive diagrams, with solutions in Python, Java, JavaScript, C++, Go, and Haskell [8][9].

For this specific problem, the useful parts are the decision flowchart (it's the tool that answers "should this be a prefix sum or a sliding window?" before you start coding) and the templates, which are generalized implementations you adapt rather than retype. The inline editor and AI assistant let you submit a working solution and get the error pinpointed, which matters here because off-by-one bugs are invisible on inspection and obvious on execution.

If you want to see how they present this exact problem before paying anything, the walkthrough page is public enough to judge the teaching style — 👉 [read AlgoMonster's prefix sum breakdown for Range Sum Query - Immutable](https://algo.monster/liteproblems/303?fpr=li69).

## AlgoMonster pricing: what the plans actually cost

This is the part where most blog coverage is out of date, so here's what to watch for. The subscribe page shows different figures during different promotions — I've seen it display `$8.25/mo` billed as `$99/year` alongside a `$189` lifetime price, and at other times `$9.92/mo` billed as `$119/year` with lifetime at `$459` [10][11]. The `$459` page also listed extras that don't appear at the lower lifetime price: one 1:1 coaching session with the founder and an internal referral for qualified candidates [10].

Some third-party write-ups claim there's no monthly plan at all [12]. That was wrong when I checked — the subscribe page has shown a Pro Monthly tier at `$18.99/month` [11]. Treat that as a reason to read the checkout page instead of any blog, including this one.

| Plan | What you get | Price | Billing | Sign up |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems — enough to judge the teaching style, not enough to prep | $0 | — | [Start on the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Monthly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks; bonus Foundation and Build-an-LLM courses | $18.99/month, and promotional rates have run lower | Monthly, cancel anytime | [Check the current monthly price](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | Everything in Pro, 12 months of access, free updates to included Pro content | Listed at $119/year ($9.92/mo); promotions have shown $99/year or less | Annual | [See the current yearly offer](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates — the higher-priced lifetime page also listed a founder coaching session and an internal referral for qualified candidates | Listed at $459 one-time; a $189 lifetime price has also been displayed during promotions | One-time | [View lifetime access pricing](https://algo.monster/subscribe?fpr=li69) |

Note that partner courses developed with outside teams, like Quant Interview Prep, are sold separately rather than bundled [10].

Which one makes sense depends on your timeline, not on the discount. If you have an interview in six weeks and you're working through patterns for the first time, a month at `$18.99` covers it, and you can decide about a longer commitment afterward. The yearly plan only wins if you expect to be prepping on and off for most of a year. Lifetime is the right call for the specific person who interviews every two to three years and hates re-subscribing — everyone else is paying for content they'll finish and shelve.

## Two things worth knowing before you buy

First, AlgoMonster is text and diagrams, with no video walkthroughs at all [8][12]. If you learn by watching someone type, that's a genuine mismatch and no pricing table fixes it. Second, the coverage of system design and behavioral interviews is introductory, so candidates targeting senior loops typically pair it with something else [12]. Both limits are documented openly by the platform's own reviewers, which is more than you get from most comparison posts.

For a problem like this one, though, the format is an advantage. Reading a prefix sum explanation and immediately editing code in the same page is faster than watching an eighteen-minute video about subtraction.

## FAQ

**Is prefix sum the expected solution for 303, or is there something better?**
Prefix sum is the expected solution. Segment trees also work and are technically more general, but for an immutable array they're overkill — you'd be paying O(n) build and O(log n) query for a problem that admits O(1) queries [4].

**Can I solve it with a hash map instead of an array?**
Yes, and some submissions do. You can store `prefix[index]` in a hash map with a `-1 → 0` sentinel and return `prefix[right] - prefix[left - 1]` [13]. It's the same complexity. The array is simpler and has better constants, so use the array unless you have a reason not to.

**Do I need to handle negative numbers?**
They're already in the sample. Since you're only ever adding and subtracting, negatives work with no special handling — which is exactly why prefix sums don't transfer cleanly to prefix min or prefix max.

**Does the array really never change?**
That's the whole premise of 303. The moment updates enter the picture you're in 307 territory and need a different structure entirely.

Prefix sums are one of maybe a dozen techniques that show up constantly across array and subarray problems. Learn it properly once — including why the extra slot exists — and 304, 560, and Product of Array Except Self all become variations instead of new problems. If you'd rather learn it alongside the rest of the pattern family than piece it together from forum posts, 👉 [AlgoMonster's free tier is the cheapest way to see whether the format works for you](https://algo.monster/subscribe?fpr=li69).
