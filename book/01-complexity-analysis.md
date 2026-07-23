# Complexity Analysis

Complexity analysis is not about attaching `O(n)` to code after writing it.

It is a way of reasoning about how the work and memory used by an algorithm grow as the input grows.

The important question is not:

> How many seconds did this program take on my laptop?

The important question is:

> What operations does the algorithm perform, how often are they performed, and what controls that number?

A timer measures one execution on one machine.

Complexity analysis explains the structure of the algorithm.

## 1. Start With the Operation

Before counting anything, I need to decide:

1. What is the input?
2. What does its size mean?
3. Which operation represents the important work?

Consider:

```c
int first(const int *array) {
    return array[0];
}
```

If the precondition is that `array` points to at least one valid `int`, this function performs one indexed access.

The number of elements stored elsewhere in the array does not matter.

Its running time is constant:

$$
\Theta(1)
$$

Now consider:

```c
#include <stddef.h>
#include <stdint.h>

int64_t sum(const int *array, size_t length) {
    int64_t total = 0;

    for (size_t i = 0; i < length; ++i) {
        total += array[i];
    }

    return total;
}
```

If `length = n`, the addition in the loop runs exactly `n` times.

The running time is:

$$
T(n) = an + b
$$

The constants `a` and `b` represent machine-level work such as incrementing `i`, checking the loop condition, reading an element, adding it, and returning.

The exact values depend on the compiler and machine.

The growth does not.

Therefore:

$$
T(n) \in \Theta(n)
$$

The important fact is not merely that there is a loop.

The important fact is that the loop performs one unit of useful work for every element.

## 2. Input Size Must Be Defined

The symbol `n` has no meaning until I define it.

For an array, `n` usually means the number of elements.

For a string, it may mean the number of characters before the null terminator.

For a graph, the input may need two variables:

- $V$: number of vertices
- $E$: number of edges

For a matrix, the input may need:

- $r$: number of rows
- $c$: number of columns

A loop over every matrix cell is:

$$
\Theta(rc)
$$

Calling it `O(n)` without saying what `n` means hides useful information.

There is another subtle case:

```c
void repeat(unsigned int value) {
    for (unsigned int i = 0; i < value; ++i) {
        /* constant work */
    }
}
```

The loop performs `value` iterations.

If I define $n = \text{value}$, the time is $\Theta(n)$.

But an integer with value $n$ needs only $\Theta(\log n)$ bits to be written in binary.

Measured against the number of input bits, the loop is exponential.

This is why input size must be stated rather than assumed.

## 3. Count Growth, Not Syntax

A loop is not automatically linear.

Two nested loops are not automatically quadratic.

I need to count how many times the core operation executes.

### A Linear Loop

```c
for (size_t i = 0; i < n; ++i) {
    work();
}
```

`work()` runs `n` times.

$$
\Theta(n)
$$

### A Fixed-Step Loop

```c
for (size_t i = 0; i < n; i += 2) {
    work();
}
```

`work()` runs about $n/2$ times.

$$
\Theta(n)
$$

Dividing the work by a constant does not change the growth class.

### A Halving Loop

Assume $n \ge 1$:

```c
for (size_t remaining = n; remaining > 1; remaining /= 2) {
    work();
}
```

After each iteration, `remaining` is roughly half its previous value.

The values look like:

$$
n,\ \frac{n}{2},\ \frac{n}{4},\ \frac{n}{8},\ldots
$$

After $k$ iterations:

$$
\frac{n}{2^k} \le 1
$$

Therefore:

$$
k \ge \log_2 n
$$

The running time is:

$$
\Theta(\log n)
$$

### A Triangular Loop

```c
for (size_t i = 0; i < n; ++i) {
    for (size_t j = 0; j <= i; ++j) {
        work();
    }
}
```

The inner loop does not run `n` times for every value of `i`.

It runs:

$$
1 + 2 + 3 + \cdots + n
$$

The total is:

$$
\frac{n(n+1)}{2}
$$

Therefore:

$$
\Theta(n^2)
$$

The conclusion comes from the sum, not from seeing two loop headers.

### Consecutive Loops

```c
for (size_t i = 0; i < n; ++i) {
    work_a();
}

for (size_t i = 0; i < n * n; ++i) {
    work_b();
}
```

The total work is:

$$
n + n^2
$$

For large `n`, the quadratic term dominates:

$$
\Theta(n^2)
$$

Consecutive costs add.

Independent nested costs usually multiply.

Dependent nested loops must be counted from their actual bounds.

## 4. From an Exact Count to a Growth Class

Suppose an algorithm performs:

$$
T(n) = 4n^2 + 7n + 12
$$

For large `n`, the $n^2$ term grows faster than the linear and constant terms.

The coefficient `4` changes the amount of work but not the type of growth.

Therefore:

$$
T(n) \in \Theta(n^2)
$$

This does not mean constants never matter in real programs.

They matter when comparing implementations, cache behavior, system calls, allocations, and small inputs.

Asymptotic analysis answers a narrower question:

> What happens to the growth rate as the input becomes large?

Common growth classes are:

| Growth | Typical name | Example |
|---|---|---|
| $\Theta(1)$ | Constant | Read `array[index]` |
| $\Theta(\log n)$ | Logarithmic | Binary search |
| $\Theta(n)$ | Linear | Scan an array |
| $\Theta(n \log n)$ | Linearithmic | Efficient comparison sorting |
| $\Theta(n^2)$ | Quadratic | Compare every pair |
| $\Theta(2^n)$ | Exponential | Explore every subset by binary choice |
| $\Theta(n!)$ | Factorial | Explore every permutation |

The difference becomes severe as `n` grows.

An algorithm that performs $n^2$ operations may be acceptable for `n = 1000`.

It is usually not acceptable for `n = 10^8`.

## 5. Big O, Big Theta, and Big Omega

These symbols describe bounds on growth.

They do not describe how happy or unhappy a particular input makes the algorithm.

### Big O: Asymptotic Upper Bound

I write:

$$
T(n) \in O(g(n))
$$

when there are positive constants $c$ and $n_0$ such that:

$$
T(n) \le c \cdot g(n)
$$

for every $n \ge n_0$.

Big O says that, beyond some point, $g(n)$ is an upper bound up to a constant factor.

If:

$$
T(n) = 3n + 5
$$

then $T(n) \in O(n)$.

It is also true that $T(n) \in O(n^2)$.

The second statement is correct but loose.

### Big Omega: Asymptotic Lower Bound

I write:

$$
T(n) \in \Omega(g(n))
$$

when there are positive constants $c$ and $n_0$ such that:

$$
T(n) \ge c \cdot g(n)
$$

for every $n \ge n_0$.

Big Omega says that the function grows at least this quickly, up to a constant factor.

If:

$$
T(n) = 3n + 5
$$

then $T(n) \in \Omega(n)$.

It is also in $\Omega(1)$, but that lower bound is loose.

### Big Theta: Asymptotically Tight Bound

I write:

$$
T(n) \in \Theta(g(n))
$$

when $g(n)$ is both an asymptotic upper bound and an asymptotic lower bound.

Equivalently, there are positive constants $c_1$, $c_2$, and $n_0$ such that:

$$
c_1g(n) \le T(n) \le c_2g(n)
$$

for every $n \ge n_0$.

If:

$$
T(n) = 3n + 5
$$

then:

$$
T(n) \in \Theta(n)
$$

Theta gives the tight growth class.

When I know a tight bound, I should prefer it.

### A Useful Language Rule

I should avoid saying:

> The algorithm is equal to $O(n)$.

Big O is a set of functions satisfying an upper-bound condition.

A more precise statement is:

> The running time is in $O(n)$.

In ordinary discussion, people often say “the algorithm is $O(n)$.”

That shorthand is common, but I should still understand the actual meaning.

## 6. Bounds Are Not Cases

Big O does not mean worst case.

Big Omega does not mean best case.

Theta does not mean average case.

The case and the bound answer different questions.

First I choose a case:

- best case
- average case under a stated distribution
- worst case

Then I bound the running-time function for that case.

For example, the worst-case running time of linear search is in $\Theta(n)$.

The phrase “worst case” chooses the inputs.

The symbol $\Theta$ gives a tight bound on the resulting function.

## 7. Best, Average, and Worst Cases

Consider linear search:

```c
#include <stdbool.h>
#include <stddef.h>

bool contains(const int *array, size_t length, int target) {
    for (size_t i = 0; i < length; ++i) {
        if (array[i] == target) {
            return true;
        }
    }

    return false;
}
```

Let one comparison with `target` be the main operation.

### Best Case

If the first element equals `target`, the function performs one comparison.

$$
\Theta(1)
$$

### Worst Case

If the target is absent, or appears only at the final position, the function performs `n` comparisons.

$$
\Theta(n)
$$

### Average Case

An average requires a probability model.

Suppose:

1. The search is known to be successful.
2. The target is equally likely to be at any of the `n` positions.

The expected number of comparisons is:

$$
\frac{1 + 2 + \cdots + n}{n}
=
\frac{n+1}{2}
$$

Therefore the average running time under this model is:

$$
\Theta(n)
$$

Now change the model.

Suppose the target is absent with probability $q$. When it is present, its position is uniform.

The expected number of comparisons becomes:

$$
q n + (1-q)\frac{n+1}{2}
$$

If $q$ is a fixed constant, this is still $\Theta(n)$, but the constant factor changes.

Average case does not mean:

> Take the best case and the worst case and divide by two.

It means:

> Compute an expectation under an explicit distribution of inputs.

Without a distribution, “average case” is incomplete.

## 8. Dry Run: Inserting Into an Array

Assume an array has enough capacity for one more element.

The first `length` positions are occupied.

To insert `value` at `position`, the elements from `position` onward must move one place to the right.

```c
#include <stdbool.h>
#include <stddef.h>

bool insert_at(
    int *array,
    size_t length,
    size_t capacity,
    size_t position,
    int value
) {
    if (position > length || length >= capacity) {
        return false;
    }

    for (size_t i = length; i > position; --i) {
        array[i] = array[i - 1];
    }

    array[position] = value;
    return true;
}
```

Take:

```txt
array    = [10, 20, 30, 40, 50, _]
length   = 5
capacity = 6
position = 2
value    = 25
```

The dry run is:

| Step | `i` | Assignment | Array state |
|---|---:|---|---|
| Start | — | — | `[10, 20, 30, 40, 50, _]` |
| 1 | 5 | `array[5] = array[4]` | `[10, 20, 30, 40, 50, 50]` |
| 2 | 4 | `array[4] = array[3]` | `[10, 20, 30, 40, 40, 50]` |
| 3 | 3 | `array[3] = array[2]` | `[10, 20, 30, 30, 40, 50]` |
| Final | — | `array[2] = 25` | `[10, 20, 25, 30, 40, 50]` |

The number of shifts is:

$$
n - p
$$

where:

- $n$ is the current length
- $p$ is the insertion position

If `position == length`, there are no shifts.

Appending into available capacity is $\Theta(1)$.

If `position == 0`, every existing element shifts.

Insertion at the front is $\Theta(n)$.

If the position is uniformly distributed from `0` through `n`, the expected number of shifts is:

$$
\frac{n}{2}
$$

The average running time under that assumption is $\Theta(n)$.

The loop moves backward for a reason.

If it moved forward, it would overwrite values before copying them.

Complexity analysis and correctness depend on understanding the same operation.

## 9. Dry Run: A Triangular Loop

Consider:

```c
for (size_t i = 0; i < n; ++i) {
    for (size_t j = 0; j <= i; ++j) {
        work();
    }
}
```

For `n = 4`:

| `i` | Values of `j` | Calls to `work()` |
|---:|---|---:|
| 0 | `0` | 1 |
| 1 | `0, 1` | 2 |
| 2 | `0, 1, 2` | 3 |
| 3 | `0, 1, 2, 3` | 4 |

Total:

$$
1 + 2 + 3 + 4 = 10
$$

For general `n`:

$$
1 + 2 + \cdots + n = \frac{n(n+1)}{2}
$$

Therefore:

$$
\Theta(n^2)
$$

A dry run exposes the sequence that must be summed.

## 10. Dry Run: Binary Search

Binary search requires a sorted array.

I can represent the current search range as a half-open interval:

$$
[low, high)
$$

`low` is included.

`high` is excluded.

```c
#include <stdbool.h>
#include <stddef.h>

bool binary_contains(
    const int *array,
    size_t length,
    int target
) {
    size_t low = 0;
    size_t high = length;

    while (low < high) {
        size_t middle = low + (high - low) / 2;

        if (array[middle] == target) {
            return true;
        }

        if (array[middle] < target) {
            low = middle + 1;
        } else {
            high = middle;
        }
    }

    return false;
}
```

Take:

```txt
array  = [3, 8, 12, 17, 23, 31, 44, 58]
target = 31
```

| Step | Range `[low, high)` | `middle` | Value | Decision |
|---|---|---:|---:|---|
| 1 | `[0, 8)` | 4 | 23 | Search right: `[5, 8)` |
| 2 | `[5, 8)` | 6 | 44 | Search left: `[5, 6)` |
| 3 | `[5, 6)` | 5 | 31 | Found |

Every unsuccessful comparison discards roughly half of the remaining range.

The best case is $\Theta(1)$ when the first middle element is the target.

The worst case is $\Theta(\log n)$.

The expression:

```c
low + (high - low) / 2
```

is preferred to:

```c
(low + high) / 2
```

because `low + high` can overflow even when both indices are valid.

The half-open interval also handles an empty array naturally:

```txt
low = 0
high = 0
```

The loop does not execute.

## 11. Time and Space Must Be Analyzed Separately

An algorithm may save time by using more memory.

It may save memory by repeating work.

Time complexity measures how the number of operations grows.

Space complexity measures how memory use grows.

I should also distinguish:

- input space: memory occupied by the input
- auxiliary space: extra memory used by the algorithm
- output space: memory required for the result

Consider:

```c
int64_t sum_recursive(const int *array, size_t length) {
    if (length == 0) {
        return 0;
    }

    return array[length - 1]
        + sum_recursive(array, length - 1);
}
```

The function visits each element once.

Its running time is:

$$
\Theta(n)
$$

But every recursive call remains on the call stack until the base case returns.

Its auxiliary space is:

$$
\Theta(n)
$$

The iterative `sum` function also takes $\Theta(n)$ time, but uses $\Theta(1)$ auxiliary space.

Ignoring the call stack would give the wrong space analysis.

## 12. Amortized Analysis

Some operations are usually cheap but occasionally expensive.

Looking only at the worst cost of one operation can hide the cost of a long sequence.

A dynamic array is the standard example.

Suppose it doubles its capacity whenever it becomes full.

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntVector;

bool vector_push(IntVector *vector, int value) {
    if (vector == NULL) {
        return false;
    }

    if (vector->size == vector->capacity) {
        size_t new_capacity = 1;

        if (vector->capacity != 0) {
            if (vector->capacity > SIZE_MAX / 2) {
                return false;
            }

            new_capacity = vector->capacity * 2;
        }

        if (new_capacity > SIZE_MAX / sizeof(*vector->data)) {
            return false;
        }

        int *new_data =
            malloc(new_capacity * sizeof(*new_data));

        if (new_data == NULL) {
            return false;
        }

        for (size_t i = 0; i < vector->size; ++i) {
            new_data[i] = vector->data[i];
        }

        free(vector->data);
        vector->data = new_data;
        vector->capacity = new_capacity;
    }

    vector->data[vector->size] = value;
    ++vector->size;
    return true;
}
```

One push can trigger allocation and copy every existing element.

If the vector currently contains `n` elements, that individual push costs:

$$
\Theta(n)
$$

But not every push performs a resize.

The capacities grow like:

```txt
1, 2, 4, 8, 16, ...
```

Over `m` successful pushes, the total number of copied elements is bounded by:

$$
1 + 2 + 4 + \cdots < 2m
$$

There are also `m` writes of the new values.

Therefore, the total work for `m` pushes is:

$$
\Theta(m)
$$

The amortized cost per push is:

$$
\frac{\Theta(m)}{m} = \Theta(1)
$$

This does not claim that every push is constant time.

It claims that the expensive pushes are rare enough that a sequence of pushes has constant cost per operation when the total is spread across the sequence.

### Aggregate View

The argument above adds the actual cost of all operations and divides by the number of operations.

This is the aggregate method.

### Accounting View

I can imagine charging each cheap push a few extra credits.

Those saved credits pay for future copies during a resize.

No probability is involved.

### Potential View

A potential function stores a numerical measure of prepaid work in the current state.

For a dynamic array that only doubles and never shrinks, one useful potential is:

$$
\Phi = 2 \cdot size - capacity
$$

after the first allocation.

Between resizes, the potential grows as elements are appended.

When a resize occurs, the stored potential drops and pays for copying the old elements.

The amortized cost is:

$$
\text{actual cost} + \Delta\Phi
$$

With an appropriate cost model, each push has amortized cost bounded by a constant.

Amortized analysis gives a guarantee over every valid sequence of operations.

Average-case analysis gives an expectation under a probability distribution.

They are not the same.

### Why the Growth Factor Matters

Suppose capacity increases by only one whenever the array fills.

The copies over `m` pushes become:

$$
1 + 2 + 3 + \cdots + (m-1)
$$

The total is:

$$
\Theta(m^2)
$$

The amortized cost per push becomes:

$$
\Theta(m)
$$

Geometric growth is what makes repeated append efficient.

## 13. C Operations Can Hide Work

Source code that occupies one line may perform work proportional to the input.

For example:

```c
size_t length = strlen(text);
```

`strlen` must scan until it finds the null terminator.

If the string length is `n`, the call is $\Theta(n)$.

This matters inside loops:

```c
for (size_t i = 0; i < strlen(text); ++i) {
    work(text[i]);
}
```

If the compiler does not safely hoist or otherwise eliminate repeated calls, `strlen` may scan the string during every iteration.

The source-level algorithm can perform quadratic work.

A clearer version stores the length:

```c
size_t length = strlen(text);

for (size_t i = 0; i < length; ++i) {
    work(text[i]);
}
```

Other operations with hidden size-dependent work include:

- `memcpy(destination, source, n)` — copies `n` bytes
- `memmove(destination, source, n)` — moves `n` bytes safely across overlap
- `strcmp(left, right)` — compares until a mismatch or null terminator
- zero-initializing a newly allocated block — touches the initialized region

Array indexing in C is constant time because the address is computed directly:

$$
\text{base address} + \text{index} \times \text{element size}
$$

That does not mean every expression using an array is constant time.

I must inspect the operation being performed.

## 14. The Computational Model Must Be Clear

Complexity analysis uses a model of computation.

For ordinary data-structure analysis, I usually treat these as constant-time operations on fixed-width machine values:

- arithmetic
- comparison
- assignment
- pointer arithmetic within a valid object
- reading or writing one array element

This model is useful, but it is an abstraction.

An integer with millions of digits cannot be added in constant time.

Disk access is not equivalent to a CPU register access.

Cache misses change real running time.

Allocation behavior depends on the allocator and system.

I do not need to include every hardware detail in every analysis.

I do need to state a different model when those details are central to the problem.

## 15. Edge Cases Are Part of the Analysis

An asymptotic answer does not prove that the program is correct.

In C, I should check at least the following cases.

### Empty Input

What happens when `length == 0`?

A loop may correctly execute zero times.

An expression such as:

```c
array[length - 1]
```

is invalid because `size_t` is unsigned and `length - 1` underflows.

### One Element

An algorithm that repeatedly halves a range should still terminate for `n == 1`.

This case often exposes incorrect interval boundaries.

### Null Pointers

A function may choose to allow `array == NULL` when `length == 0`, because it never dereferences the pointer.

That is an interface decision and should be documented.

If `length > 0`, a null array pointer is invalid.

### Capacity and Length

For an array-backed structure:

$$
0 \le size \le capacity
$$

An insertion must reject a position greater than `size`.

It must not write beyond `capacity`.

### Integer Overflow

These expressions can overflow:

```c
capacity * 2
count * sizeof(*array)
low + high
```

Overflow checks are correctness requirements.

They do not change the normal asymptotic class, but omitting them can make the implementation invalid for large inputs.

Signed integer overflow in C is undefined behavior.

Using a wider accumulator reduces risk but does not make overflow impossible.

### Allocation Failure

`malloc` can return `NULL`.

A data structure must preserve its old valid state when growth fails.

### Invalid Preconditions

Binary search on an unsorted array is not a meaningful “worst case.”

It violates the algorithm's precondition.

Complexity cases range over valid inputs unless stated otherwise.

### Duplicates

If a search returns any matching element, duplicates may not affect its asymptotic complexity.

If it must return the first or last match, the operation and stopping rule change.

### Recursion Depth

A recursive algorithm may have acceptable time complexity and still overflow the call stack.

Space and implementation limits must be analyzed separately.

## 16. Common Misconceptions

### “Big O Means Worst Case”

No.

Big O means asymptotic upper bound.

I can give a Big O bound for a best-case, average-case, worst-case, or amortized running-time function.

### “Big Omega Means Best Case”

No.

Big Omega means asymptotic lower bound.

The worst-case running time of linear search is in $\Omega(n)$ and in $O(n)$, so it is in $\Theta(n)$.

### “Two Nested Loops Always Mean $O(n^2)$”

No.

The bounds may be logarithmic, triangular, dependent on another variable, or constant.

I must count iterations.

### “$O(2n)$ Is Faster Than $O(n)$”

Both describe the same asymptotic growth class.

The factor `2` may matter in practice, but it does not create a different Big O class.

### “If Something Is $O(n)$, It Cannot Be $O(n^2)$”

It can.

Big O is an upper bound.

A linear function is also bounded above by a quadratic function for sufficiently large `n`.

The tight statement is $\Theta(n)$.

### “Average Case Is Halfway Between Best and Worst”

No.

Average case requires probabilities for the valid inputs.

### “Amortized Means Probable”

No.

Amortized analysis does not require randomness.

It bounds the total cost of a sequence.

### “Logarithm Bases Change the Complexity”

For constant bases greater than one:

$$
\log_a n = \frac{\log_b n}{\log_b a}
$$

The difference is a constant factor.

Therefore all constant bases give $\Theta(\log n)$.

### “$O(1)$ Means Instant”

No.

It means the operation count does not grow with the chosen input size.

A constant-time operation can still have a large constant.

### “Asymptotically Better Always Means Faster”

No.

For small inputs, constants and hardware behavior can dominate.

Asymptotic analysis predicts growth, not an exact crossover point.

### “One Line of C Is One Operation”

No.

`strlen`, `memcpy`, allocation, comparison, and called functions may hide substantial work.

### “Recursive Code Uses Constant Space if It Allocates No Heap Memory”

No.

Each active call usually consumes stack space.

## 17. A Repeatable Analysis Method

When I analyze an operation, I will use this order:

1. Define the input and its size variables.
2. State the valid-input preconditions.
3. Choose the operation to count.
4. Count how often it executes.
5. Write a sum or recurrence when inspection is not enough.
6. Separate best, average, worst, and amortized claims.
7. Simplify to a tight asymptotic bound.
8. Analyze auxiliary space separately.
9. Check empty input, one element, overflow, allocation failure, and invalid indices.
10. Dry-run a small example to test the reasoning.

This method is more reliable than memorizing rules about loops.

## 18. Practice Questions

For every question:

1. Define the input size.
2. Identify the counted operation.
3. Give a tight bound when possible.
4. State any assumptions.
5. Analyze auxiliary space.

### Question 1: Two Input Sizes

```c
for (size_t i = 0; i < rows; ++i) {
    for (size_t j = 0; j < columns; ++j) {
        matrix[i][j] = 0;
    }
}
```

Analyze the time in terms of `rows` and `columns`.

When, if ever, is it reasonable to call this $\Theta(n^2)$?

### Question 2: A Loop That Looks Quadratic

```c
for (size_t i = 1; i < n; i *= 2) {
    for (size_t j = 0; j < i; ++j) {
        work();
    }
}
```

Count the total calls to `work()`.

Then identify the overflow and termination risk in the C loop control.

Rewrite the loop so the intended analysis remains valid near the limit of `size_t`.

### Question 3: Repeated Halving

```c
size_t count = 0;

for (size_t x = n; x > 0; x /= 2) {
    ++count;
}
```

Dry-run the code for:

- `n = 0`
- `n = 1`
- `n = 8`
- `n = 13`

Find a tight bound and explain why the exact iteration count changes at powers of two.

### Question 4: Pair Counting

```c
for (size_t i = 0; i < n; ++i) {
    for (size_t j = i + 1; j < n; ++j) {
        compare(array[i], array[j]);
    }
}
```

Derive the exact number of calls to `compare`.

What property of the pairs prevents double-counting?

### Question 5: Hidden String Work

```c
for (size_t i = 0; i < strlen(text); ++i) {
    if (text[i] == 'x') {
        return true;
    }
}
```

Give best- and worst-case bounds under a source-level cost model where each `strlen` scans the string.

Then rewrite the function and analyze it again.

Explain why compiler optimization should not replace algorithmic reasoning in the source-level design.

### Question 6: Linear Search With a Distribution

An array has `n` elements.

The target is absent with probability $1/3$.

If present, it is equally likely to occur first at any one of the `n` positions.

Find the expected number of comparisons made by linear search.

Give the exact expression and the tight asymptotic bound.

### Question 7: Dynamic Growth by 50 Percent

A dynamic array grows from capacity `c` to:

$$
c + \lceil c/2 \rceil
$$

when full.

Does append still have $\Theta(1)$ amortized cost?

Prove the answer by bounding the total number of copied elements over `m` pushes.

### Question 8: Dynamic Growth by a Constant

A dynamic array increases capacity by `100` whenever full.

Analyze:

- the worst cost of one append
- the total cost of `m` appends
- the amortized cost per append

Explain why adding `100` is asymptotically different from multiplying capacity by a constant greater than one.

### Question 9: Recursive Space

```c
bool contains_recursive(
    const int *array,
    size_t length,
    int target
) {
    if (length == 0) {
        return false;
    }

    if (array[0] == target) {
        return true;
    }

    return contains_recursive(array + 1, length - 1, target);
}
```

Analyze best- and worst-case time.

Analyze auxiliary space.

Would tail-call optimization be guaranteed by the C language?

### Question 10: Binary Search Invariant

For the half-open binary search in this chapter, state a precise loop invariant involving `[low, high)`.

Show that:

1. The invariant is true before the first iteration.
2. Each branch preserves it.
3. The interval becomes smaller.
4. Termination with `low == high` proves absence.

Then modify the algorithm to return the first occurrence of `target` in an array containing duplicates.

### Question 11: A Loose Bound

An algorithm has exact cost:

$$
7n \log_2 n + 40n + 900
$$

Which statements are true?

- $T(n) \in O(n^2)$
- $T(n) \in O(n \log n)$
- $T(n) \in \Omega(n)$
- $T(n) \in \Omega(n \log n)$
- $T(n) \in \Theta(n \log n)$

For every true statement, say whether it is tight or loose.

### Question 12: Insert and Delete Sequence

An initially empty dynamic array doubles when full.

It never shrinks.

Perform this sequence:

1. Push `n` elements.
2. Delete the last element `n` times.
3. Push `n` elements again.

Assume deleting the last element is constant time and does not clear unused storage.

Find a tight bound for the complete sequence.

How would the analysis change if every deletion reduced capacity by exactly one?

### Question 13: Space Definitions

An algorithm receives an array of `n` integers, allocates a second array of `n` integers, fills it, and returns it.

State:

- input space
- output space
- auxiliary space

Explain how the answer depends on whether the returned array is classified as output or temporary workspace.

### Question 14: Overflow-Safe Allocation

Write a C function that allocates space for `count` elements of size `element_size`.

It must reject multiplication overflow before calling `malloc`.

Analyze the normal running time and auxiliary space.

Explain why $\Theta(1)$ complexity does not mean the function is automatically safe.

### Question 15: Construct the Cases

For each operation below, construct a valid input that produces its best case and one that produces its worst case:

- linear search
- binary search
- insertion into an array with spare capacity
- append to a doubling dynamic array

Do not stop at naming the bound.

Show the actual operation count for a small dry run.

## 19. Final Check

Before moving on, I should be able to explain:

- why complexity begins with operations rather than loop-counting shortcuts
- why `n` must be defined
- why $O$, $\Theta$, and $\Omega$ are bounds rather than cases
- why average case requires a probability model
- why amortized analysis does not mean average case
- why one dynamic-array append can be $\Theta(n)$ while append is still $\Theta(1)$ amortized
- why hidden library calls and recursion matter
- why edge cases can break correct-looking C code without changing its asymptotic label

If I cannot explain those points with a dry run, I do not yet understand the chapter.
