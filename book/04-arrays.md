# Arrays

An array stores a fixed number of same-typed objects in one contiguous sequence.

That short definition explains most of its power.

Because the elements have equal size and occupy consecutive positions, the address of any valid element can be calculated directly from:

- the base address
- the index
- the element size

That gives constant-time indexed access.

The same layout also gives:

- predictable traversal
- good spatial locality
- compact storage without a pointer in every element
- simple interoperability with low-level memory operations

The tradeoff is equally structural.

Contiguity means there is no empty gap in the middle where a new logical element can simply appear.

Insertion and deletion inside the sequence require movement if order must be preserved.

A fixed array also cannot increase the number of element slots in its own object.

This chapter studies ordinary fixed-capacity arrays.

The next chapter will study dynamic arrays, where a separate owning object can replace its allocation when more capacity is required.

Unless a section explicitly says otherwise, code in this chapter targets standard C17 rather than compiler extensions or C23-only syntax.

Character arrays obey the same storage rules, but null termination, text input, encodings, and string-library contracts belong to the later Strings chapter.

The memory chapter introduced some of C's pointer and lifetime rules.

This chapter restates the subset needed for array algorithms so that each contract and dry run remains understandable on its own.

## Part I — Array Foundations

### 1. The Problem an Array Solves

Suppose I need to store five temperatures.

I could create five unrelated variables:

```c
int temperature_0;
int temperature_1;
int temperature_2;
int temperature_3;
int temperature_4;
```

That representation is awkward.

To process every value, I would need to mention every variable separately.

An array gives the values one shared structure:

```c
int temperatures[5] = {18, 20, 21, 19, 17};
```

Now one index selects an element:

```c
temperatures[0]
temperatures[1]
temperatures[2]
temperatures[3]
temperatures[4]
```

One loop can process the entire sequence:

```c
for (size_t i = 0; i < 5; ++i) {
    printf("%d\n", temperatures[i]);
}
```

The array solves two related problems:

1. It stores a collection of same-typed values as one object.
2. It gives every element a numeric position that can be computed efficiently.

The index is not merely a label.

It is part of the address calculation.

### 2. An Array Is One Object Containing Element Objects

Consider:

```c
int values[4] = {10, 20, 30, 40};
```

`values` is an array object.

Its type is:

```txt
array of 4 int
```

It contains four element objects:

```txt
values
+-----------+-----------+-----------+-----------+
| values[0] | values[1] | values[2] | values[3] |
|    10     |    20     |    30     |    40     |
+-----------+-----------+-----------+-----------+
```

The array object and its first element begin at the same address, but they are not the same typed object.

These expressions illustrate the distinction:

```c
&values
&values[0]
```

They have the same numeric starting location on an ordinary printed address display.

Their types differ:

```txt
&values       pointer to array of 4 int
&values[0]    pointer to int
```

Pointer arithmetic therefore differs:

```txt
&values + 1       advances past the complete 4-int array
&values[0] + 1    advances to the second int
```

The address alone is not the whole meaning.

The pointed-to type determines the step.

### 3. The Core Array Invariant

For an array of `n` elements:

1. Every element has the same type.
2. The elements are contiguous.
3. Valid indices are `0` through `n - 1` when `n > 0`.
4. Element order matches index order.
5. The array's number of slots is fixed for that array object's lifetime.

If the element type is `T`, the conceptual layout is:

```txt
base
 |
 v
+----------+----------+----------+-----+----------+
| element 0| element 1| element 2| ... |element n-1|
+----------+----------+----------+-----+----------+
   T size     T size     T size            T size
```

There is no per-element link field.

There is no hidden per-element capacity marker.

The position itself gives the relationship.

#### Logical Length Is a Separate Invariant

A physical array may provide more slots than the application currently uses.

```c
int storage[8];
size_t length = 3;
```

The physical capacity is `8`.

The logical length is `3`.

Conceptually:

```txt
+----+----+----+----+----+----+----+----+
| 10 | 20 | 30 | ?? | ?? | ?? | ?? | ?? |
+----+----+----+----+----+----+----+----+
|<-- logical elements -->|
|<------------- physical capacity ------------->|
```

The application-level invariant is:

```txt
length <= capacity
```

Only indices in:

```txt
[0, length)
```

are logical elements.

The remaining slots may exist physically without containing values the data structure is allowed to read.

This distinction drives append, insertion, deletion, and every fixed-capacity interface later in the chapter.

### 4. Array Declarations Encode Element Type and Count

The declaration:

```c
int values[10];
```

means:

```txt
values is an array of 10 int
```

These are different types:

```c
int integers[10];
double measurements[10];
char bytes[10];
```

Even if two array types happen to occupy the same number of bytes, their element types and permitted operations differ.

#### The Count Must Be Positive for an Ordinary Fixed Array

Standard C does not define an ordinary zero-length array:

```c
int empty[0]; /* Not standard C. */
```

Some compilers accept zero-length arrays as an extension.

Portable code should not rely on that extension.

An empty logical sequence can instead be represented by:

```txt
pointer = NULL
length = 0
```

or by a nonempty storage array whose logical length is zero.

#### The Element Type Must Be Complete

The compiler must know each element's size to lay out the array.

This is invalid while `struct Node` is incomplete:

```c
struct Node;
struct Node nodes[10];
```

A pointer array is possible because pointer size is known:

```c
struct Node *nodes[10];
```

After the structure definition is complete, an array of actual nodes can be declared.

### 5. Initialization Determines the First Valid Values

This automatic array is uninitialized:

```c
void example(void) {
    int values[4];
}
```

Its elements have indeterminate values.

Reading them before initialization is wrong.

#### Full Initializer

```c
int values[4] = {10, 20, 30, 40};
```

Every element is explicitly initialized.

#### Partial Initializer

```c
int values[4] = {10, 20};
```

The remaining elements are initialized as zero:

```txt
[10, 20, 0, 0]
```

This rule makes:

```c
int values[4] = {0};
```

a common way to initialize every integer element to zero.

The first element is explicitly initialized to zero.

Every omitted element is also initialized as zero.

#### Inferred Count

```c
int values[] = {10, 20, 30, 40};
```

The compiler infers an array of four `int`.

This prevents a manually written count from disagreeing with the initializer.

#### Designated Initializers

```c
int values[8] = {
    [2] = 25,
    [6] = 70,
};
```

The result is:

```txt
[0, 0, 25, 0, 0, 0, 70, 0]
```

Unspecified elements are zero-initialized.

Designators are useful for sparse tables with meaningful fixed indices.

They should not be confused with a sparse data structure.

The full physical array still exists.

#### Static Storage Is Zero-Initialized

Objects with static storage duration are initialized before program execution.

If no explicit initializer is given:

```c
static int values[100];
```

every element initially has value zero.

That does not apply to an uninitialized automatic array inside an ordinary block.

### 6. Indexing Comes From Address Arithmetic

For a pointer or array expression `base` and integer index `i`, C defines:

```c
base[i]
```

as:

```c
*(base + i)
```

Suppose:

```txt
base address = B
element size = s
index = i
```

Then the conceptual byte address is:

$$
B + i \cdot s
$$

For:

```c
int values[4] = {10, 20, 30, 40};
```

if `sizeof(int) == 4` and the base address is conceptually `1000`, the element addresses are:

| Index | Address calculation | Conceptual address |
|---:|---|---:|
| `0` | `1000 + 0 * 4` | `1000` |
| `1` | `1000 + 1 * 4` | `1004` |
| `2` | `1000 + 2 * 4` | `1008` |
| `3` | `1000 + 3 * 4` | `1012` |

The actual addresses are chosen by the implementation.

The constant stride is guaranteed by array contiguity.

No scan through earlier elements is needed.

Therefore indexed access takes:

$$
\Theta(1)
$$

under the ordinary fixed-width machine model.

#### Constant Time Does Not Mean Free

An access may still involve:

- multiplication or scaled addressing
- address translation
- a cache miss
- a page fault
- bounds checks added by surrounding code

The operation count does not grow with array length.

That is the asymptotic claim.

### 7. Why Indices Begin at Zero

The index represents an offset measured in elements from the base.

The first element is zero elements away:

```txt
address of element 0 = base + 0 * element_size
```

The second is one element away:

```txt
address of element 1 = base + 1 * element_size
```

The last element of an `n`-element array is:

```txt
n - 1 elements away
```

This makes a half-open range natural:

```txt
[0, n)
```

It contains exactly `n` integer indices:

```txt
0, 1, 2, ..., n - 1
```

The range length is:

```txt
end - begin = n - 0 = n
```

Zero-based indexing is therefore not arbitrary decoration.

It matches displacement from the base address and half-open range arithmetic.

### 8. Valid Bounds and the One-Past Position

For:

```c
int values[4];
```

valid element indices are:

```txt
0, 1, 2, 3
```

This is out of bounds:

```c
values[4]
```

The pointer:

```c
values + 4
```

may be formed as the one-past pointer.

It is useful as an exclusive endpoint:

```c
for (
    int *current = values;
    current != values + 4;
    ++current
) {
    use(*current);
}
```

The one-past pointer must not be dereferenced:

```c
int value = *(values + 4); /* Undefined behavior. */
```

This mistake is not repaired by the fact that another object may happen to occupy the next address.

Array bounds are defined by the array object, not by whether mapped memory happens to exist.

### 9. `sizeof` Can Recover a Local Array's Physical Size

When the array expression has not decayed to a pointer:

```c
int values[10];
```

then:

```c
sizeof values
```

is the total byte size of the array.

And:

```c
sizeof values[0]
```

is the byte size of one element.

The element count is:

```c
sizeof values / sizeof values[0]
```

A readable macro is:

```c
#define ARRAY_COUNT(array) \
    (sizeof(array) / sizeof((array)[0]))
```

Used with an actual array:

```c
int values[] = {10, 20, 30, 40};
size_t count = ARRAY_COUNT(values);
```

`count` is `4`.

#### The Macro Is Dangerous With a Pointer

```c
int *pointer = values;
size_t wrong = ARRAY_COUNT(pointer);
```

The macro divides:

```txt
size of pointer / size of pointed-to int
```

That is not the allocation length.

It may even produce a plausible small number and hide the mistake.

The macro is safe only where the argument is known to be an actual array expression.

#### Prefer Inferred Counts Near Initializers

```c
static const int primes[] = {2, 3, 5, 7, 11};

static const size_t prime_count =
    sizeof primes / sizeof primes[0];
```

The data and its derived count remain synchronized.

### 10. Arrays and Pointers Are Different

This declaration creates an array:

```c
int values[4];
```

This declaration creates a pointer:

```c
int *pointer;
```

The array contains four `int` objects.

The pointer contains one pointer value.

The array has fixed storage as part of its own object.

The pointer may be reassigned to designate different objects.

```c
pointer = values;
pointer = &another_value;
```

The array name cannot be assigned:

```c
values = pointer; /* Invalid. */
```

#### Why They Seem Similar

In most expressions, an array expression is converted to a pointer to its first element.

```c
int *pointer = values;
```

The conversion result is:

```c
&values[0]
```

This is array-to-pointer conversion, often called decay.

The conversion does not transform the array object into a pointer object.

It produces a pointer value for that expression.

#### Important Non-Decaying Contexts

An array does not undergo the ordinary conversion when it is:

- the operand of `sizeof`
- the operand of unary `&`
- a string literal used to initialize a character array

For example:

```c
sizeof values
```

measures the whole array.

```c
&values
```

produces a pointer to the whole array.

The string-literal exception lets this declaration initialize an array rather than merely produce a pointer:

```c
char word[] = "cat";
```

`word` contains four `char` elements:

```txt
'c', 'a', 't', '\0'
```

The array mechanics in this chapter apply normally.

Calling a character array a C string additionally requires a terminating null character inside its valid extent.

Literal mutability, string length, text input, and string-library operations are deferred to the Strings chapter.

Understanding exactly where decay occurs prevents many false assumptions about length.

### 11. Function Parameters Do Not Receive Array Length

These parameter declarations are adjusted to pointer parameters:

```c
void process(int values[]);
void process(int values[100]);
void process(int *values);
```

For an ordinary function parameter, all three declare `values` as `int *`.

Inside the function:

```c
sizeof values
```

measures the pointer parameter.

It does not recover the caller's array size.

This function is wrong:

```c
size_t element_count(int values[]) {
    return sizeof values / sizeof values[0];
}
```

The robust ordinary interface carries length explicitly:

```c
void process(
    const int *values,
    size_t length
);
```

The pointer says where the first element is.

The length says how many logical elements the function may inspect.

Unless an operation explicitly documents overlap support, this chapter requires each output parameter to identify a separate object outside the input range, backing storage, and descriptor.

That conservative rule prevents an output write from corrupting unread input or metadata.

If mutation may use spare slots, capacity must also be passed:

```c
bool insert_at(
    int *values,
    size_t *length,
    size_t capacity,
    size_t index,
    int value
);
```

The pointer alone contains neither length nor capacity.

### 12. Pointer-Plus-Length Is a Range Contract

Consider:

```c
long long sum(
    const int *values,
    size_t length
);
```

A clear contract can state:

- if `length > 0`, `values` points to the first of at least `length` readable `int`
- if `length == 0`, `values` may be null
- the function does not modify the elements
- the caller keeps ownership
- every left-to-right partial sum, including the final sum, is representable in `long long`

Implementation:

```c
long long sum(
    const int *values,
    size_t length
) {
    long long total = 0;

    for (size_t i = 0; i < length; ++i) {
        total += values[i];
    }

    return total;
}
```

The loop executes zero times when `length == 0`.

No dereference occurs.

Requiring only the final mathematical sum to fit would be insufficient.

Earlier positive and negative values could make an intermediate prefix overflow even if later values bring the final result back into range.

Without the partial-sum precondition or checked addition, signed overflow in the accumulator would be undefined behavior.

The function must still avoid operations that are invalid on null, such as forming an endpoint with pointer arithmetic:

```c
const int *end = values + length;
```

Even adding zero to a null pointer is not a portable way to form an array range.

Index-based loops make the permitted empty-null contract easy to preserve.

#### Output Parameters Need Independent Validation

```c
bool maximum(
    const int *values,
    size_t length,
    int *maximum_out
) {
    if (
        maximum_out == NULL
        || length == 0
        || values == NULL
    ) {
        return false;
    }

    int maximum_value = values[0];

    for (size_t i = 1; i < length; ++i) {
        if (values[i] > maximum_value) {
            maximum_value = values[i];
        }
    }

    *maximum_out = maximum_value;
    return true;
}
```

The output is written only after success is established.

On failure, `*maximum_out` remains unchanged.

### 13. `const` Expresses Read-Only Array Access

This function may read but not modify elements through its parameter:

```c
size_t count_equal(
    const int *values,
    size_t length,
    int target
);
```

This function may modify elements:

```c
void fill(
    int *values,
    size_t length,
    int value
);
```

Use `const` for read-only array views.

It:

- documents intent
- lets the compiler reject writes through that parameter
- permits callers to pass genuinely const arrays

It does not guarantee that no other alias can modify the same storage.

```c
int values[3] = {1, 2, 3};
const int *read_view = values;
int *write_view = values;

write_view[0] = 99;
printf("%d\n", read_view[0]); /* 99 */
```

The `const` pointer provides a restricted access path.

It does not freeze the underlying object when writable aliases exist.

### 14. Storage Duration Controls Array Lifetime

#### Automatic Array

```c
void work(void) {
    int values[100];
}
```

The array's lifetime ends when execution leaves the block.

Returning `values` would return a pointer to dead storage:

```c
int *bad_array(void) {
    int values[10] = {0};
    return values; /* Wrong. */
}
```

#### Static Array

```c
static int values[100];
```

The array exists for the whole program execution.

A pointer to it can outlive the declaring function, although shared mutable state may create design and concurrency problems.

#### Allocated Array

```c
int *values = NULL;

if (count <= SIZE_MAX / sizeof *values) {
    values = malloc(count * sizeof *values);
}
```

The allocation is not an array declaration, but the returned storage can be used to hold an array of `int` when size calculation and initialization are correct.

A null result must be handled before dereference.

Its lifetime ends at `free` or successful `realloc`.

The dynamic-array chapter will build ownership and growth around allocated storage.

#### A View Does Not Extend Lifetime

If a structure stores:

```c
int *data;
size_t length;
```

it does not keep the pointed-to array alive automatically.

The view becomes dangling when the caller's storage lifetime ends.

Every non-owning array interface must document that relationship.

## Part II — Fixed-Capacity Sequences

### 15. Fixed Capacity and Logical Length

A raw C array knows its physical element count only where its array type is still available.

An application often tracks logical length separately:

```c
enum { CAPACITY = 8 };

int storage[CAPACITY];
size_t length = 0;
```

The invariant is:

```txt
0 <= length <= CAPACITY
```

Because `size_t` is unsigned, the lower bound is automatic for representable values.

The meaningful condition is:

```txt
length <= capacity
```

#### State Examples

Empty:

```txt
length = 0
capacity = 8
logical sequence = []
```

Partially full:

```txt
length = 3
capacity = 8
logical sequence = [10, 20, 30]
```

Full:

```txt
length = 8
capacity = 8
```

Append is possible only when:

```txt
length < capacity
```

Insertion at any position is possible only when:

```txt
index <= length
and
length < capacity
```

Access and removal require:

```txt
index < length
```

The strict and non-strict inequalities are different for a reason.

Insertion may occur immediately after the last logical element.

Access may not.

### 16. Put the Invariant Beside the Array

Passing `data`, `length`, and `capacity` separately works, but it is easy to mix values from different arrays.

A small non-owning view keeps the related state together:

| Representation | Owns storage? | Extent or capacity can change? | What happens when full? | Can existing element addresses be invalidated by growth? |
|:---|:---:|:---:|:---|:---:|
| Raw fixed array, such as `int values[8]` | storage is the array object itself | no | application decides how many slots are logical | no growth exists |
| Fixed-capacity view in this chapter | no; it borrows a raw array | logical length changes, capacity does not | append and insertion fail | no growth exists |
| Owning dynamic array in the next chapter | yes | capacity can change by replacing storage | may attempt allocation and relocation | yes |

The middle representation may look like a dynamic array because it has a length and append operation.

It is not one.

It manages only which prefix of fixed borrowed storage is active.

```c
typedef struct {
    int *data;
    size_t length;
    size_t capacity;
} IntArray;
```

This structure does not allocate memory.

It does not own the storage.

It only describes:

- where the storage begins
- how many elements are logically present
- how many element slots physically exist

A useful validity predicate is:

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

static bool int_array_is_valid(const IntArray *array) {
    if (array == NULL) {
        return false;
    }

    if (array->length > array->capacity) {
        return false;
    }

    if (array->capacity > SIZE_MAX / sizeof *array->data) {
        return false;
    }

    if (array->capacity == 0) {
        return array->data == NULL && array->length == 0;
    }

    return array->data != NULL;
}
```

The multiplication guard may look unnecessary because this view does not allocate.

It still matters.

A capacity that cannot be represented as a byte count cannot describe one valid C array object of that many `int` elements.

The predicate therefore captures the representable-object invariant too.

This chapter uses one normalized empty representation:

```txt
data = NULL
length = 0
capacity = 0
```

Another interface could permit:

```txt
data = non-null
length = 0
capacity = 0
```

Neither convention is universally mandatory.

The convention must be explicit and consistent.

#### Initialize Empty Storage

```c
static bool int_array_init(
    IntArray *array,
    int *storage,
    size_t capacity
) {
    if (array == NULL) {
        return false;
    }

    if (capacity > SIZE_MAX / sizeof *storage) {
        return false;
    }

    if ((capacity == 0 && storage != NULL) ||
        (capacity > 0 && storage == NULL)) {
        return false;
    }

    array->data = storage;
    array->length = 0;
    array->capacity = capacity;
    return true;
}
```

Example:

```c
int storage[8];
IntArray values;

if (!int_array_init(&values, storage, 8)) {
    /* Handle invalid arguments. */
}
```

The elements of `storage` have indeterminate values at this point because it is an uninitialized automatic array.

That is safe only because `values.length` is zero.

No operation may read `storage[0]` as a logical element until an operation writes it and increases the length.

#### Wrap Existing Logical Elements

Sometimes the storage already contains values:

```c
static bool int_array_wrap(
    IntArray *array,
    int *storage,
    size_t length,
    size_t capacity
) {
    if (array == NULL || length > capacity) {
        return false;
    }

    if (capacity > SIZE_MAX / sizeof *storage) {
        return false;
    }

    if ((capacity == 0 && storage != NULL) ||
        (capacity > 0 && storage == NULL)) {
        return false;
    }

    array->data = storage;
    array->length = length;
    array->capacity = capacity;
    return true;
}
```

Usage:

```c
int storage[6] = {10, 20, 30};
IntArray values;

if (!int_array_wrap(&values, storage, 3, 6)) {
    /* Handle invalid arguments. */
}
```

Only the first three slots are logical elements.

The zeros in the remaining slots came from partial initialization, but they are still outside the logical sequence.

#### The View Does Not Own Anything

This is invalid:

```c
static IntArray bad_factory(void) {
    int local[8];
    IntArray result;
    int_array_init(&result, local, 8);
    return result; /* result.data immediately dangles */
}
```

The returned structure is copied successfully.

The automatic array it points into no longer exists.

The caller must keep the underlying array alive longer than every view into it.

That ownership rule is as important as `length <= capacity`.

### 17. Direct Access and Update

Reading one known index requires only validation and one indexed access:

```c
static bool int_array_get(
    const IntArray *array,
    size_t index,
    int *out_value
) {
    if (!int_array_is_valid(array) ||
        out_value == NULL ||
        index >= array->length) {
        return false;
    }

    *out_value = array->data[index];
    return true;
}
```

Updating is similar:

```c
static bool int_array_set(
    IntArray *array,
    size_t index,
    int value
) {
    if (!int_array_is_valid(array) || index >= array->length) {
        return false;
    }

    array->data[index] = value;
    return true;
}
```

Both operations are:

```txt
Theta(1) time
Theta(1) auxiliary space
```

#### Failure Leaves Observable State Alone

`int_array_get` writes `*out_value` only after every check succeeds.

This lets a caller distinguish failure without losing an earlier value:

```c
int value = 99;

if (!int_array_get(&values, 100, &value)) {
    /* value is still 99 */
}
```

`int_array_set` checks the index before writing the array.

On failure, no logical value, length, or capacity changes.

This is a small example of failure atomicity:

```txt
either the operation succeeds,
or the observable data structure remains unchanged
```

#### Access Is Not Search

These questions are different:

```txt
What is at index 7?
Where is the value 7?
```

The first is direct access.

The second requires a search unless another structure stores the answer.

Arrays make index-to-address conversion constant time.

They do not automatically make value-to-index lookup constant time.

### 18. Traversal Is Repeated Direct Access

To print every logical element:

```c
static void int_array_print(const IntArray *array) {
    if (!int_array_is_valid(array)) {
        return;
    }

    putchar('[');

    for (size_t i = 0; i < array->length; ++i) {
        if (i > 0) {
            fputs(", ", stdout);
        }

        printf("%d", array->data[i]);
    }

    puts("]");
}
```

For length `n`, the loop performs `n` visits.

Therefore:

```txt
Theta(n) time
Theta(1) auxiliary space
```

Direct access is constant time per element.

Visiting all elements is still linear because there are `n` elements to visit.

#### Empty Traversal

When `length == 0`, the loop condition is false before the first iteration:

```txt
i = 0
0 < 0 is false
```

No special loop branch is required.

Good half-open ranges make empty cases fall out naturally.

#### Pointer Traversal

The same logical traversal could be written:

```c
for (const int *cursor = array->data;
     cursor != array->data + array->length;
     ++cursor) {
    printf("%d\n", *cursor);
}
```

But this version needs care for the normalized empty view.

When `data == NULL` and `length == 0`, even forming:

```c
array->data + 0
```

is not a useful portable foundation: pointer arithmetic is defined relative to an actual array object, and a null pointer does not point into one.

The index-based loop avoids forming such an expression when the range is empty.

For this interface, the index form is clearer.

### 19. Linear Search

An unsorted array may contain the target anywhere.

The direct method checks values from left to right:

```c
static bool int_array_find(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return false;
    }

    for (size_t i = 0; i < array->length; ++i) {
        if (array->data[i] == target) {
            *out_index = i;
            return true;
        }
    }

    return false;
}
```

The function returns the first matching index.

For:

```txt
[8, 4, 8, 2]
```

searching for `8` returns index `0`, not index `2`.

That duplicate policy is part of the contract.

#### Dry Run

Search for `7` in:

```txt
[4, 9, 7, 3]
```

| Step | `i` | `data[i]` | Equal to `7`? |
|---:|---:|---:|:---|
| 1 | 0 | 4 | no |
| 2 | 1 | 9 | no |
| 3 | 2 | 7 | yes |

The function writes `2` and returns `true`.

Index `3` is never inspected.

#### Case Analysis

For length `n`:

- best case: the first element matches, so `Theta(1)`
- worst case: the target is absent or last, so `Theta(n)`
- average case under a stated distribution: `Theta(n)`

The average claim needs assumptions.

For example, if a successful target is equally likely to be at each index, the expected comparisons are:

```txt
(1 + 2 + ... + n) / n
= n(n + 1) / (2n)
= (n + 1) / 2
= Theta(n)
```

If almost every query asks for the first element, the observed average could instead be close to constant.

Input distribution matters.

### 20. Append Writes at the Logical End

Append adds one value after the existing logical sequence:

```c
static bool int_array_append(IntArray *array, int value) {
    if (!int_array_is_valid(array) ||
        array->length == array->capacity) {
        return false;
    }

    array->data[array->length] = value;
    ++array->length;
    return true;
}
```

The order of mutation matters:

1. write the new logical value
2. publish the larger logical length

If length were increased first and later code failed before the write, the invariant would claim that an unwritten slot was logical.

This simple function has no intervening failure, but write-then-publish is still the right habit.

#### Dry Run

Initial state:

```txt
data     = [10, 20, 30, ?, ?]
length   = 3
capacity = 5
```

Append `40`:

```txt
write data[length]
write data[3] = 40
```

Then:

```txt
length = 4
```

Final state:

```txt
data     = [10, 20, 30, 40, ?]
length   = 4
capacity = 5
```

One write and one increment are independent of `n`:

```txt
Theta(1) time
Theta(1) auxiliary space
```

This fixed-capacity append has no resizing cost.

When the array is full, it fails.

The next chapter will examine dynamic arrays, where a full append may allocate a larger block and copy elements.

### 21. Insertion Requires Opening a Gap

Insertion at index `k` means:

```txt
old elements [0, k) stay where they are
new value becomes element k
old elements [k, n) move one position right
length becomes n + 1
```

The operation requires:

```txt
k <= length
length < capacity
```

Implementation:

```c
static bool int_array_insert(
    IntArray *array,
    size_t index,
    int value
) {
    if (!int_array_is_valid(array) ||
        index > array->length ||
        array->length == array->capacity) {
        return false;
    }

    for (size_t i = array->length; i > index; --i) {
        array->data[i] = array->data[i - 1];
    }

    array->data[index] = value;
    ++array->length;
    return true;
}
```

#### Why the Shift Runs Backward

Insert `20` at index `1`:

```txt
before:

index      0    1    2    3    4
         +----+----+----+----+----+
data     | 10 | 30 | 40 | 50 | ?? |
         +----+----+----+----+----+
length = 4
```

Required moves:

```txt
data[4] = data[3]   move 50
data[3] = data[2]   move 40
data[2] = data[1]   move 30
data[1] = 20        fill gap
```

State progression:

```txt
[10, 30, 40, 50, 50]   after moving 50
[10, 30, 40, 40, 50]   after moving 40
[10, 30, 30, 40, 50]   after moving 30
[10, 20, 30, 40, 50]   after writing 20
```

Every source is read before that source is overwritten.

A forward shift would destroy needed values:

```txt
data[2] = data[1]   gives [10, 30, 30, 50, ?]
data[3] = data[2]   now copies the new 30, losing 40
```

The direction follows from overlap.

When moving a range toward higher indices, copy backward.

#### Boundary Cases

Insert at the front:

```txt
index = 0
all n old elements move
Theta(n)
```

Insert in the middle:

```txt
n - index elements move
Theta(n - index)
```

Insert at the end:

```txt
index = length
zero old elements move
Theta(1)
```

Worst-case time is `Theta(n)`.

Best-case time is `Theta(1)`.

The exact operation count is governed by distance from the logical end, not by capacity.

### 22. Deletion Closes a Gap

Removing index `k` while preserving order means:

```txt
save old element k if the caller wants it
move old elements [k + 1, n) one position left
decrease length
```

```c
static bool int_array_remove(
    IntArray *array,
    size_t index,
    int *out_value
) {
    if (!int_array_is_valid(array) ||
        out_value == NULL ||
        index >= array->length) {
        return false;
    }

    int removed = array->data[index];

    for (size_t i = index; i < array->length - 1; ++i) {
        array->data[i] = array->data[i + 1];
    }

    --array->length;
    *out_value = removed;
    return true;
}
```

Because `index < length`, the successful path guarantees `length >= 1`.

Therefore:

```c
array->length - 1
```

cannot underflow in the loop condition.

#### Dry Run

Remove index `1`:

```txt
before:

index      0    1    2    3    4
         +----+----+----+----+----+
data     | 10 | 20 | 30 | 40 | 50 |
         +----+----+----+----+----+
length = 5
```

Save:

```txt
removed = 20
```

Shift left:

```txt
data[1] = data[2]   [10, 30, 30, 40, 50]
data[2] = data[3]   [10, 30, 40, 40, 50]
data[3] = data[4]   [10, 30, 40, 50, 50]
```

Then:

```txt
length = 4
out_value = 20
```

Logical result:

```txt
[10, 30, 40, 50]
```

The stale `50` in physical slot `4` is outside `[0, length)`.

It is not a duplicate logical element.

#### Why the Shift Runs Forward

The source for each assignment is one position to the right:

```txt
destination i <- source i + 1
```

Copying from low indices to high indices reads each source before it can be overwritten.

When moving an overlapping range toward lower indices, copy forward.

#### Complexity

Remove first:

```txt
n - 1 moves
Theta(n)
```

Remove last:

```txt
zero moves
Theta(1)
```

Remove index `k`:

```txt
n - k - 1 moves
Theta(n - k)
```

The final expression ignores constant differences, as asymptotic notation should.

### 23. Clearing the Sequence Does Not Require Erasing Storage

Logical clear is:

```c
static bool int_array_clear(IntArray *array) {
    if (!int_array_is_valid(array)) {
        return false;
    }

    array->length = 0;
    return true;
}
```

This is `Theta(1)`.

It changes which elements belong to the data structure.

It does not overwrite old bytes.

After:

```txt
before logical sequence = [10, 20, 30]
after logical sequence  = []
```

the storage may still physically contain:

```txt
[10, 20, 30, ...]
```

Reading those slots through the cleared data structure would violate its logical invariant.

#### Logical Removal and Secure Erasure Are Different Requirements

If values contain secrets, merely changing `length` may be insufficient.

Secure erasure is a separate system-level concern involving:

- whether the compiler may remove apparently unused stores
- which erasure primitive the implementation provides
- copies in registers or other buffers
- crash dumps and swap
- the lifetime of the underlying storage

Do not claim that `length = 0` securely erases data.

Also do not pay `Theta(n)` to zero every slot unless the contract actually requires physical clearing.

### 24. `memmove` Encodes Overlapping Movement

The insertion shift can also use a library operation:

```c
#include <string.h>

size_t count = array->length - index;

memmove(
    &array->data[index + 1],
    &array->data[index],
    count * sizeof array->data[0]
);
```

Deletion can use:

```c
size_t count = array->length - index - 1;

memmove(
    &array->data[index],
    &array->data[index + 1],
    count * sizeof array->data[0]
);
```

`memmove` is designed for overlapping source and destination ranges.

`memcpy` is not.

Using `memcpy` when the ranges overlap gives undefined behavior, even if a particular build appears to copy in the desired direction.

#### Zero-Byte Calls Still Need Sensible Pointer Reasoning

For a valid nonempty storage array, the insertion and deletion expressions above remain within the array or its one-past boundary.

But do not generalize this into:

```c
memmove(NULL, NULL, 0);
```

as the foundation of an interface.

Library contracts and compiler optimizations around invalid pointers deserve conservative code.

It is usually clearer to skip a bulk operation when the count is zero:

```c
if (count > 0) {
    memmove(destination, source, count * sizeof *source);
}
```

#### Byte-Count Overflow Must Be Ruled Out

The expression:

```c
count * sizeof array->data[0]
```

must be representable in `size_t`.

The view invariant:

```txt
capacity <= SIZE_MAX / sizeof(int)
```

and:

```txt
count <= length <= capacity
```

together establish that safety.

The byte multiplication is not an isolated trick.

It depends on the data structure invariant.

### 25. Reversal Exchanges Symmetric Pairs

To reverse:

```txt
[10, 20, 30, 40, 50]
```

exchange:

```txt
index 0 with index 4
index 1 with index 3
```

The middle element remains where it is.

```c
static void reverse_range(
    int *data,
    size_t begin,
    size_t end
) {
    while (end - begin > 1) {
        --end;

        int temporary = data[begin];
        data[begin] = data[end];
        data[end] = temporary;

        ++begin;
    }
}
```

The range is half-open:

```txt
[begin, end)
```

The function assumes that range is valid for `data`.

The public operation establishes that precondition:

```c
static bool int_array_reverse(IntArray *array) {
    if (!int_array_is_valid(array)) {
        return false;
    }

    if (array->length > 1) {
        reverse_range(array->data, 0, array->length);
    }

    return true;
}
```

#### Why `end - begin > 1` Is Useful

The loop does not compute:

```c
end - 1
```

until it knows the range has at least two elements.

That prevents unsigned underflow for an empty range.

#### Dry Run

Start:

```txt
[10, 20, 30, 40, 50]
 begin               end
   0                  5
```

First exchange:

```txt
--end gives 4
swap indices 0 and 4
[50, 20, 30, 40, 10]
begin becomes 1
```

Second exchange:

```txt
--end gives 3
swap indices 1 and 3
[50, 40, 30, 20, 10]
begin becomes 2
```

Now:

```txt
end - begin = 3 - 2 = 1
```

Stop.

Every element is involved in at most one exchange:

```txt
Theta(n) time
Theta(1) auxiliary space
```

### 26. Rotation by Three Reversals

A left rotation by `k` moves the first `k` elements to the end while preserving both groups' internal order.

Example:

```txt
left rotate [1, 2, 3, 4, 5, 6, 7] by 3
result      [4, 5, 6, 7, 1, 2, 3]
```

Let:

```txt
A = first k elements
B = remaining elements
```

The input is:

```txt
A B
```

The desired output is:

```txt
B A
```

Use:

```txt
reverse A       -> reverse(A) B
reverse B       -> reverse(A) reverse(B)
reverse all     -> B A
```

Implementation:

```c
static bool int_array_rotate_left(
    IntArray *array,
    size_t shift
) {
    if (!int_array_is_valid(array)) {
        return false;
    }

    if (array->length < 2) {
        return true;
    }

    shift %= array->length;

    if (shift == 0) {
        return true;
    }

    reverse_range(array->data, 0, shift);
    reverse_range(array->data, shift, array->length);
    reverse_range(array->data, 0, array->length);
    return true;
}
```

The modulo makes rotations equivalent:

```txt
k
k + n
k + 2n
```

It is performed only after the `length < 2` check, so there is no remainder-by-zero operation.

#### Dry Run

Input:

```txt
[1, 2, 3, 4, 5, 6, 7]
shift = 3
```

Reverse `[0, 3)`:

```txt
[3, 2, 1, 4, 5, 6, 7]
```

Reverse `[3, 7)`:

```txt
[3, 2, 1, 7, 6, 5, 4]
```

Reverse `[0, 7)`:

```txt
[4, 5, 6, 7, 1, 2, 3]
```

Three linear reversals are still linear:

```txt
Theta(n) time
Theta(1) auxiliary space
```

A temporary-array rotation can also be `Theta(n)` time, but needs `Theta(n)` additional storage.

Equal asymptotic time does not imply equal space behavior.

### 27. Sorted Order Creates Stronger Search Options

An unsorted array only tells us where elements are stored.

A sorted array adds an invariant:

```txt
for every valid i < j:
data[i] <= data[j]
```

That invariant lets one comparison discard half of the remaining candidate range.

#### Lower Bound

The lower bound of `target` is the first index whose value is not less than `target`.

If every value is smaller, the lower bound is `length`.

```c
static bool int_array_lower_bound(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return false;
    }

    size_t low = 0;
    size_t high = array->length;

    while (low < high) {
        size_t middle = low + (high - low) / 2;

        if (array->data[middle] < target) {
            low = middle + 1;
        } else {
            high = middle;
        }
    }

    *out_index = low;
    return true;
}
```

The search range is half-open:

```txt
[low, high)
```

Its invariant is:

```txt
all indices before low contain values < target
all indices at or after high contain values >= target
```

The unknown candidate region lies between them.

When `low == high`, no unknown region remains.

That position is the answer.

#### Why the Middle Formula Is Written This Way

This common expression can overflow:

```c
(low + high) / 2
```

The safer form is:

```c
low + (high - low) / 2
```

Because `low <= high`, the subtraction is valid, and the addition cannot exceed `high`.

#### Dry Run With Duplicates

Find the lower bound of `4`:

```txt
data = [1, 2, 4, 4, 4, 9]
```

| Step | `low` | `high` | `middle` | value | Decision |
|---:|---:|---:|---:|---:|:---|
| 1 | 0 | 6 | 3 | 4 | `high = 3` |
| 2 | 0 | 3 | 1 | 2 | `low = 2` |
| 3 | 2 | 3 | 2 | 4 | `high = 2` |

Now `low == high == 2`.

The first `4` is at index `2`.

#### Find the First Equal Element

Lower bound gives a deterministic duplicate policy:

```c
static bool int_array_binary_find_first(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return false;
    }

    size_t index = 0;

    if (!int_array_lower_bound(array, target, &index)) {
        return false;
    }

    if (index == array->length ||
        array->data[index] != target) {
        return false;
    }

    *out_index = index;
    return true;
}
```

The result output remains unchanged when the target is absent.

#### Binary Search Has a Precondition

These functions do not verify that the array is sorted.

Checking sortedness would take `Theta(n)` and erase the `Theta(log n)` search advantage.

The caller must establish and preserve the sorted-order invariant.

Calling binary search on:

```txt
[10, 2, 7, 4]
```

does not create undefined behavior by itself, but the result has no useful contract.

The algorithm is only correct under its stated precondition.

#### Complexity

Each iteration reduces the candidate range by about half:

```txt
n
n / 2
n / 4
n / 8
...
1
```

The number of halvings is logarithmic:

```txt
Theta(log n) time
Theta(1) auxiliary space
```

For an empty array, lower bound returns index `0` without reading storage.

### 28. Upper Bound Completes Duplicate-Aware Search

The upper bound of `target` is the first index whose value is greater than `target`.

```c
static bool int_array_upper_bound(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return false;
    }

    size_t low = 0;
    size_t high = array->length;

    while (low < high) {
        size_t middle = low + (high - low) / 2;

        if (array->data[middle] <= target) {
            low = middle + 1;
        } else {
            high = middle;
        }
    }

    *out_index = low;
    return true;
}
```

For:

```txt
[1, 2, 4, 4, 4, 9]
```

and target `4`:

```txt
lower bound = 2
upper bound = 5
```

Therefore the equal range is:

```txt
[2, 5)
```

and the number of occurrences is:

```txt
5 - 2 = 3
```

The pair of bounds answers several precise questions:

- membership: `lower < length && data[lower] == target`
- first equal index: `lower`
- one-past-last equal index: `upper`
- occurrence count: `upper - lower`
- insertion before existing equals: insert at `lower`
- insertion after existing equals: insert at `upper`

“Use binary search” is incomplete when duplicates are possible.

The desired boundary must be named.

#### Sorted Insertion Is Not Logarithmic

Suppose lower bound locates an insertion position in `Theta(log n)`.

The fixed array must still open a physical gap.

That can move `Theta(n)` elements.

Total time is:

```txt
Theta(log n) + Theta(n)
= Theta(n)
```

Binary search improves the comparison count used to locate the position.

It does not remove the contiguity cost.

### 29. Whole-Array Operations Are Not Built-In Value Operations

C does not permit direct assignment between array objects:

```c
int source[4] = {1, 2, 3, 4};
int destination[4];

destination = source; /* Invalid. */
```

It also does not compare contents with:

```c
if (source == destination) {
    /* This does not compare all elements. */
}
```

In most expressions, both arrays convert to pointers to their first elements.

The comparison asks whether those addresses are equal.

Two distinct arrays with identical values normally have different addresses.

#### Copy Element Semantics Explicitly

For `int`:

```c
for (size_t i = 0; i < 4; ++i) {
    destination[i] = source[i];
}
```

Or, when the source and destination ranges do not overlap:

```c
memcpy(destination, source, sizeof source);
```

If overlap is possible, use `memmove`.

#### Compare Logical Values Explicitly

```c
static bool int_ranges_equal(
    const int *left,
    const int *right,
    size_t length
) {
    if (length == 0) {
        return true;
    }

    if (left == NULL || right == NULL) {
        return false;
    }

    for (size_t i = 0; i < length; ++i) {
        if (left[i] != right[i]) {
            return false;
        }
    }

    return true;
}
```

This defines equality in terms of `int` values.

#### Why `memcmp` Is Not Universal Semantic Equality

`memcmp` compares bytes in object representations.

Semantic equality may differ from byte equality for types involving:

- padding bytes in structures
- multiple representations of a value
- pointer fields whose addresses differ while pointed-to values are equivalent
- floating-point values such as positive and negative zero

For an array of a structure:

```c
typedef struct {
    char tag;
    int value;
} Item;
```

the implementation may place padding between `tag` and `value`.

Two `Item` values can have equal members while padding bytes differ.

Member-by-member comparison expresses the intended meaning.

#### A Struct Containing an Array Can Be Assigned

This is valid:

```c
typedef struct {
    int values[4];
} FourInts;

FourInts a = {{1, 2, 3, 4}};
FourInts b = a;
```

Structure assignment copies every member, including array members.

That copy is still member-wise in its semantics.

If an element contains an owning pointer, copying the pointer does not deep-copy its allocation.

The outer array does not solve ownership inside its elements.

### 30. Half-Open Ranges Simplify Subarray Reasoning

A subarray view can be described by:

```txt
[begin, end)
```

where:

```txt
0 <= begin <= end <= length
```

Its number of elements is:

```txt
end - begin
```

Examples:

```txt
[0, length)       complete logical array
[0, 0)            empty prefix
[length, length)  empty suffix
[2, 5)            indices 2, 3, 4
```

Adjacent ranges meet without overlap:

```txt
[0, k) and [k, n)
```

That is why the rotation proof could name two clean groups.

#### Safe Reverse Traversal

This loop is wrong for unsigned `size_t`:

```c
for (size_t i = length - 1; i >= 0; --i) {
    use(data[i]);
}
```

Problems:

1. `length - 1` underflows when `length == 0`.
2. `i >= 0` is always true for an unsigned type.
3. decrementing zero wraps to `SIZE_MAX`.

Use the one-past position:

```c
for (size_t i = length; i > 0; --i) {
    use(data[i - 1]);
}
```

For `length == 0`, the body runs zero times.

For `length == 4`, visited indices are:

```txt
3, 2, 1, 0
```

The counter represents how many elements remain in the prefix, rather than trying to represent a negative stopping index.

#### Function Parameter Bounds Are Preconditions

This:

```c
void process(int data[10]);
```

is adjusted to a pointer parameter.

It does not require the caller to pass ten elements.

C also permits:

```c
void process_at_least_ten(int data[static 10]);
```

The `static 10` form states a caller precondition: on each call, `data` must provide access to the first element of an array containing at least ten elements.

It is not an automatic runtime bounds check.

Violating that precondition gives the function no valid basis for its promised access.

For general data structures, explicit pointer-plus-length interfaces usually make the range visible at the call site and inside the implementation.

### 31. Unordered Removal Trades Order for Constant Time

Stable removal shifts later elements left.

If order does not matter, replace the removed element with the last logical element:

```c
static bool int_array_remove_unordered(
    IntArray *array,
    size_t index,
    int *out_value
) {
    if (!int_array_is_valid(array) ||
        out_value == NULL ||
        index >= array->length) {
        return false;
    }

    int removed = array->data[index];
    size_t last = array->length - 1;

    if (index != last) {
        array->data[index] = array->data[last];
    }

    --array->length;
    *out_value = removed;
    return true;
}
```

Example:

```txt
before = [10, 20, 30, 40, 50]
remove index 1

copy last logical element into index 1
[10, 50, 30, 40, 50]

decrease length
logical result = [10, 50, 30, 40]
```

Time:

```txt
Theta(1)
```

The result is not:

```txt
[10, 30, 40, 50]
```

This is not a faster implementation of the same contract.

It is a different operation with a weaker ordering guarantee.

That tradeoff is useful in:

- unordered entity collections
- unordered work pools where scheduling order has no meaning
- sets represented by a dense array plus another lookup structure

It is invalid when clients depend on relative order or stable indices.

### 32. Stable Compaction Uses Read and Write Indices

Suppose I want to remove every occurrence of a value while preserving the order of everything retained.

Repeatedly calling stable single-element removal can take quadratic time:

```txt
find a match
shift the suffix
find another match
shift another suffix
...
```

A one-pass compaction avoids repeated movement:

```c
static bool int_array_remove_all(
    IntArray *array,
    int target,
    size_t *out_removed
) {
    if (!int_array_is_valid(array) || out_removed == NULL) {
        return false;
    }

    size_t write = 0;

    for (size_t read = 0; read < array->length; ++read) {
        if (array->data[read] != target) {
            array->data[write] = array->data[read];
            ++write;
        }
    }

    size_t removed = array->length - write;
    array->length = write;
    *out_removed = removed;
    return true;
}
```

#### The Preserved-Prefix Invariant

Before each iteration with a given `read`:

```txt
data[0, write)
```

contains, in original order, exactly the retained values from the old range:

```txt
old_data[0, read)
```

Also:

```txt
write <= read
```

So every destination is at or before its source.

The operation never overwrites an element that a future iteration still needs to inspect.

#### Dry Run

Remove every `2`:

```txt
input = [2, 7, 2, 9, 4, 2]
```

| `read` | value | action | `write` after | retained prefix |
|---:|---:|:---|---:|:---|
| 0 | 2 | skip | 0 | `[]` |
| 1 | 7 | write at 0 | 1 | `[7]` |
| 2 | 2 | skip | 1 | `[7]` |
| 3 | 9 | write at 1 | 2 | `[7, 9]` |
| 4 | 4 | write at 2 | 3 | `[7, 9, 4]` |
| 5 | 2 | skip | 3 | `[7, 9, 4]` |

Final:

```txt
length = 3
removed = 3
logical sequence = [7, 9, 4]
```

Complexity:

```txt
Theta(n) time
Theta(1) auxiliary space
```

The same pattern can:

- retain values satisfying a predicate
- move all nonzero values toward the front
- remove invalid records
- deduplicate a sorted array

The predicate changes.

The read/write invariant remains.

#### Deduplicate a Sorted Array

For sorted input, equal values are adjacent:

```c
static bool int_array_deduplicate_sorted(IntArray *array) {
    if (!int_array_is_valid(array)) {
        return false;
    }

    if (array->length < 2) {
        return true;
    }

    size_t write = 1;

    for (size_t read = 1; read < array->length; ++read) {
        if (array->data[read] != array->data[write - 1]) {
            array->data[write] = array->data[read];
            ++write;
        }
    }

    array->length = write;
    return true;
}
```

Input:

```txt
[1, 1, 1, 3, 3, 8]
```

Output:

```txt
[1, 3, 8]
```

The sorted precondition makes comparison with the last retained value sufficient.

Without sortedness, a duplicate may occur anywhere earlier, and constant-extra-space stable deduplication has a different cost.

## Part III — Reusable Array Patterns

### 33. Opposing Indices Implement a Two-Pointer Partition

Some problems maintain one index at each end.

As an example, partition integers so negative values appear before nonnegative values.

Order within each group is not preserved:

```c
static bool int_array_partition_negative(
    IntArray *array,
    size_t *out_nonnegative_begin
) {
    if (!int_array_is_valid(array) ||
        out_nonnegative_begin == NULL) {
        return false;
    }

    size_t left = 0;
    size_t right = array->length;

    while (left < right) {
        while (left < right && array->data[left] < 0) {
            ++left;
        }

        while (left < right && array->data[right - 1] >= 0) {
            --right;
        }

        if (left < right) {
            int temporary = array->data[left];
            array->data[left] = array->data[right - 1];
            array->data[right - 1] = temporary;

            ++left;
            --right;
        }
    }

    *out_nonnegative_begin = left;
    return true;
}
```

The active regions are:

```txt
[0, left)       already known negative
[left, right)   not fully classified
[right, n)      already known nonnegative
```

Each iteration shrinks the unknown region.

When `left == right`, classification is complete.

#### Dry Run

Input:

```txt
[3, -2, 5, -7, 0, -1]
```

Initial:

```txt
left = 0
right = 6
```

Right scan stops at `-1`.

Swap indices `0` and `5`:

```txt
[-1, -2, 5, -7, 0, 3]
```

Left scan advances over `-2` and stops at `5`.

Right scan moves over `3` and `0`, then stops at `-7`.

Swap `5` and `-7`:

```txt
[-1, -2, -7, 5, 0, 3]
```

The partition boundary is index `3`.

Complexity:

```txt
Theta(n) time
Theta(1) auxiliary space
```

Contrast this with stable compaction.

This pattern is commonly called “two pointers” even when the implementation stores integer indices.

It moves fewer values than stable compaction and uses constant space, but it does not preserve original order.

### 34. Sliding Windows Reuse Work Between Neighboring Ranges

Suppose I need the largest sum of exactly `width` adjacent values.

A direct approach recomputes every window:

```txt
sum data[0, width)
sum data[1, width + 1)
sum data[2, width + 2)
...
```

For roughly `n` windows and `width` additions per window:

```txt
Theta(n * width)
```

Neighboring windows overlap.

When moving one step:

```txt
remove the value leaving on the left
add the value entering on the right
```

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

static bool checked_add_intmax(
    intmax_t left,
    intmax_t right,
    intmax_t *out_result
) {
    if (out_result == NULL ||
        (right > 0 && left > INTMAX_MAX - right) ||
        (right < 0 && left < INTMAX_MIN - right)) {
        return false;
    }

    *out_result = left + right;
    return true;
}

static bool checked_subtract_intmax(
    intmax_t left,
    intmax_t right,
    intmax_t *out_result
) {
    if (out_result == NULL ||
        (right > 0 && left < INTMAX_MIN + right) ||
        (right < 0 && left > INTMAX_MAX + right)) {
        return false;
    }

    *out_result = left - right;
    return true;
}

static bool max_fixed_window_sum(
    const int *data,
    size_t length,
    size_t width,
    intmax_t *out_sum
) {
    if (out_sum == NULL ||
        width == 0 ||
        width > length ||
        (length > 0 && data == NULL)) {
        return false;
    }

    intmax_t current = 0;

    for (size_t i = 0; i < width; ++i) {
        intmax_t next = 0;

        if (!checked_add_intmax(current, data[i], &next)) {
            return false;
        }

        current = next;
    }

    intmax_t best = current;

    for (size_t right = width; right < length; ++right) {
        intmax_t after_removal = 0;
        intmax_t next = 0;

        if (!checked_subtract_intmax(
                current,
                data[right - width],
                &after_removal
            ) ||
            !checked_add_intmax(
                after_removal,
                data[right],
                &next
            )) {
            return false;
        }

        current = next;

        if (current > best) {
            best = current;
        }
    }

    *out_sum = best;
    return true;
}
```

`intmax_t` can represent every standard signed integer type, so every `int` converts to it safely.

It is still finite.

The checked helpers reject any intermediate addition or subtraction that would exceed its range.

The output remains unchanged if such a check fails.

#### Dry Run

Input:

```txt
data = [4, -1, 2, 10, -3, 5]
width = 3
```

First window:

```txt
[4, -1, 2] sum = 5
best = 5
```

Slide right:

```txt
remove 4, add 10
[-1, 2, 10] sum = 11
best = 11
```

Slide:

```txt
remove -1, add -3
[2, 10, -3] sum = 9
best = 11
```

Slide:

```txt
remove 2, add 5
[10, -3, 5] sum = 12
best = 12
```

Every element enters once and leaves at most once:

```txt
Theta(n) time
Theta(1) auxiliary space
```

#### Variable Windows Need a Monotonic Reason

A common pattern expands the right edge and shrinks the left edge while a condition is violated.

That is not automatically correct.

For example, shrinking a window until its sum is at most a limit behaves predictably when all values are nonnegative:

```txt
expanding cannot decrease the sum
shrinking cannot increase the sum
```

Negative values break that monotonic relationship.

A variable-window algorithm must state what property makes moving an edge irreversible and safe.

“Use sliding window” is not a proof.

### 35. Prefix Sums Move Work From Queries to Construction

Suppose the underlying values do not change and I need many range-sum queries.

Build:

```txt
prefix[0] = 0
prefix[i + 1] = prefix[i] + data[i]
```

For:

```txt
data = [5, 2, 7, 3]
```

the prefix array is:

```txt
prefix = [0, 5, 7, 14, 17]
```

Each entry means:

```txt
prefix[i] = sum of data[0, i)
```

Then:

```txt
sum of data[left, right)
= prefix[right] - prefix[left]
```

For `[1, 4)`:

```txt
prefix[4] - prefix[1]
= 17 - 5
= 12
```

which is:

```txt
2 + 7 + 3
```

#### Construction

```c
static bool build_prefix_sums(
    const int *data,
    size_t length,
    intmax_t *prefix,
    size_t prefix_capacity
) {
    if (prefix == NULL ||
        prefix_capacity < length ||
        prefix_capacity - length < 1 ||
        (length > 0 && data == NULL)) {
        return false;
    }

    prefix[0] = 0;

    for (size_t i = 0; i < length; ++i) {
        intmax_t next = 0;

        if (!checked_add_intmax(prefix[i], data[i], &next)) {
            return false;
        }

        prefix[i + 1] = next;
    }

    return true;
}
```

The readable `data[0, length)` range and writable `prefix[0, length + 1)` range must not overlap.

Without that rule, writing an early prefix entry could destroy an input value that a later iteration still needs.

Why not simply test:

```c
prefix_capacity < length + 1
```

Because `length + 1` could overflow.

The split check establishes that the capacity contains at least `length` slots and at least one additional slot without overflowing the requested count.

The checked helper rejects an unrepresentable prefix.

If that happens, entries already constructed remain valid, the failing entry and later entries are not published, and the function returns `false`.

This construction function deliberately provides a valid-prefix failure guarantee rather than full output preservation.

#### Query

```c
static bool prefix_range_sum(
    const intmax_t *prefix,
    size_t value_length,
    size_t left,
    size_t right,
    intmax_t *out_sum
) {
    if (prefix == NULL ||
        out_sum == NULL ||
        left > right ||
        right > value_length) {
        return false;
    }

    return checked_subtract_intmax(
        prefix[right],
        prefix[left],
        out_sum
    );
}
```

The checked subtraction preserves `*out_sum` if the mathematical range sum is not representable in `intmax_t`.

Complexities:

```txt
construction = Theta(n) time and Theta(n) extra space
each query  = Theta(1) time
```

Without preprocessing, each arbitrary range sum takes `Theta(range length)`.

Prefix sums are worthwhile when enough later queries reuse the construction work.

#### Updates Change the Tradeoff

If `data[k]` changes by `delta`, every prefix entry after `k` changes:

```txt
prefix[k + 1]
prefix[k + 2]
...
prefix[n]
```

A plain prefix array therefore has:

```txt
Theta(n) point update
Theta(1) range query
```

Later data structures such as Fenwick trees and segment trees choose different update/query tradeoffs.

### 36. Difference Arrays Batch Range Updates

Prefix sums accumulate values from left to right.

A difference array records changes at boundaries.

Suppose an initially zero array of length `n` must receive many operations:

```txt
add delta to every index in [left, right)
```

For a difference array with `n + 1` slots:

```txt
difference[left]  += delta
difference[right] -= delta
```

After every update, one prefix pass reconstructs final values.

Example with length `6`:

```txt
add 5 to [1, 4)
add 2 to [3, 6)
```

Boundary changes:

```txt
difference[1] += 5
difference[4] -= 5

difference[3] += 2
difference[6] -= 2
```

Difference state:

```txt
[0, 5, 0, 2, -5, 0, -2]
```

Prefix accumulation over the first six positions:

```txt
[0, 5, 5, 7, 2, 2]
```

This matches applying both range updates directly.

For `q` updates:

```txt
direct repeated updates = potentially Theta(q * n)
difference boundaries   = Theta(q)
final reconstruction    = Theta(n)
total                   = Theta(q + n)
```

This technique requires:

- all relevant updates to be known before final materialization, or a contract for when reconstruction occurs
- an accumulator type wide enough for combined deltas
- valid half-open endpoints
- space for the one-past boundary at index `n`

It is not a substitute for a dynamic online query structure.

### 37. Frequency Arrays Exchange Generality for Direct Counting

If every value lies in a small known integer domain, the value itself can index a count array.

Suppose scores are guaranteed to be in:

```txt
0 through 100
```

```c
enum { SCORE_COUNT = 101 };

size_t frequencies[SCORE_COUNT] = {0};

for (size_t i = 0; i < length; ++i) {
    int score = scores[i];

    if (score < 0 || score >= SCORE_COUNT) {
        /* Contract violation or input error. */
    } else {
        ++frequencies[(size_t)score];
    }
}
```

Then:

```txt
frequencies[x]
```

gives the count of score `x` in constant time.

Construction is:

```txt
Theta(n + domain size)
```

including zero-initialization.

Space is:

```txt
Theta(domain size)
```

The technique is excellent for a small dense domain.

It is wasteful or impossible for:

- arbitrary `int` values
- enormous sparse key ranges
- values that cannot safely become array indices

Negative values must be rejected or mapped through a proven offset.

If the domain is `[minimum, maximum]`, the mapped index:

```txt
value - minimum
```

needs overflow-safe arithmetic and a range check before conversion to `size_t`.

The array makes counting direct only because the domain contract makes indexing valid.

## Part IV — Shapes, Layout, and Locality

### 38. Multidimensional Arrays Are Arrays of Arrays

Consider:

```c
enum {
    ROWS = 3,
    COLUMNS = 4
};

int matrix[ROWS][COLUMNS];
```

Read the type from the identifier outward:

```txt
matrix is an array of 3 elements
each element is an array of 4 int
```

The three row elements are:

```txt
matrix[0]
matrix[1]
matrix[2]
```

Each row has type:

```txt
array of 4 int
```

Each scalar element is:

```txt
matrix[row][column]
```

Conceptual layout:

```txt
matrix

row 0: [0][0] [0][1] [0][2] [0][3]
row 1: [1][0] [1][1] [1][2] [1][3]
row 2: [2][0] [2][1] [2][2] [2][3]
```

C stores the row arrays contiguously in row-major order.

The last index varies fastest.

The sizes are:

```c
sizeof matrix
    == ROWS * COLUMNS * sizeof matrix[0][0]

sizeof matrix[0]
    == COLUMNS * sizeof matrix[0][0]
```

There is no hidden padding between array elements.

If the scalar element type itself is a structure with padding, that padding is part of every element's size and therefore part of the stride.

#### Nested Initialization

Braces can make row structure visible:

```c
int initialized[][3] = {
    {1, 2, 3},
    {4}
};
```

The first extent is inferred as `2`.

The column extent must still be known as `3`, because it determines every row's type and stride.

The second row is partially initialized:

```txt
[4, 0, 0]
```

The full matrix is:

```txt
[
    [1, 2, 3],
    [4, 0, 0]
]
```

Omitted scalar elements follow the same zero-initialization rule as a one-dimensional partially initialized array.

#### Address Reasoning Uses the Row Type

In most expressions:

```c
matrix
```

converts to a pointer to its first row:

```txt
pointer to array of 4 int
```

Its effective type is:

```c
int (*)[COLUMNS]
```

Therefore:

```txt
matrix + 1
```

advances by one complete row, not by one `int`.

Then:

```c
matrix[row][column]
```

is conceptually:

```c
*(*(matrix + row) + column)
```

The first addition selects a row.

The second selects an element within that row.

#### `int **` Is a Different Representation

This is not the type of a two-dimensional array:

```c
int **
```

An `int **` points to an `int *`.

It often represents:

```txt
pointer to a table of row pointers
each row pointer may identify a separate allocation
```

Those rows may:

- have different lengths
- live at unrelated addresses
- require separate allocation and cleanup

A real:

```c
int matrix[ROWS][COLUMNS]
```

contains row arrays directly and decays to:

```c
int (*)[COLUMNS]
```

Casting one representation to the other does not make the row stride correct.

### 39. Passing a Matrix Requires the Column Extent

With a compile-time column count:

```c
enum { COLUMNS = 4 };

static void zero_rows(
    int matrix[][COLUMNS],
    size_t rows
) {
    for (size_t row = 0; row < rows; ++row) {
        for (size_t column = 0;
             column < COLUMNS;
             ++column) {
            matrix[row][column] = 0;
        }
    }
}
```

The parameter is adjusted to:

```c
int (*matrix)[COLUMNS]
```

The compiler needs `COLUMNS` to calculate:

```txt
matrix + row
```

because that addition advances by one row object.

#### Runtime Column Extent With a VLA Parameter Type

C17 permits:

```c
static void fill_matrix(
    size_t rows,
    size_t columns,
    int matrix[rows][columns],
    int value
) {
    for (size_t row = 0; row < rows; ++row) {
        for (size_t column = 0;
             column < columns;
             ++column) {
            matrix[row][column] = value;
        }
    }
}
```

After parameter adjustment, the important type is:

```c
int (*matrix)[columns]
```

The dimensions must appear before the parameter that uses them.

This interface adopts the simple precondition:

```txt
rows > 0
columns > 0
```

The caller must provide a compatible rectangular array.

A flat pointer-plus-shape interface is a better fit when zero-row or zero-column logical matrices must be represented.

VLA support is optional in conforming C11 and C17 implementations.

Portable libraries that target implementations without VLA support should use a flat representation or a fixed column bound.

#### Do Not Flatten Through the First Row Pointer

It is tempting to write:

```c
int *cursor = &matrix[0][0];

for (size_t i = 0; i < ROWS * COLUMNS; ++i) {
    use(cursor[i]);
}
```

The physical bytes are contiguous, but strict pointer-arithmetic rules are expressed in terms of array objects.

`&matrix[0][0]` points into the first row array.

Teaching it as an unrestricted `ROWS * COLUMNS` flat `int` range crosses row subarray boundaries and is not the soundest portable model.

Prefer:

- nested row and column indexing
- pointer-to-row traversal
- an actually flat array representation
- representation copying through appropriate byte operations when that is the real intent

Physical contiguity and the bounds of a particular typed pointer operation are related but not identical claims.

### 40. Flat Matrices Make the Stride Explicit

A matrix can instead use one flat array:

```txt
data points to rows * columns int elements
```

Element `(row, column)` maps to:

```txt
row * columns + column
```

For:

```txt
rows = 3
columns = 4
row = 2
column = 1
```

the index is:

```txt
2 * 4 + 1 = 9
```

#### Overflow-Safe Index Calculation

```c
static bool flat_matrix_index(
    size_t rows,
    size_t columns,
    size_t row,
    size_t column,
    size_t *out_index
) {
    if (out_index == NULL ||
        row >= rows ||
        column >= columns) {
        return false;
    }

    if (columns > 0 && row > SIZE_MAX / columns) {
        return false;
    }

    size_t base = row * columns;

    if (column > SIZE_MAX - base) {
        return false;
    }

    *out_index = base + column;
    return true;
}
```

Given valid bounds:

```txt
row < rows
column < columns
```

and a separately established valid total element count, some overflow checks can be derived from the representation invariant.

Keeping a checked helper makes the reasoning local and explicit.

#### Validate the Complete Storage Requirement

Before claiming a flat storage block represents `rows * columns` `int` elements:

```c
static bool flat_matrix_element_count(
    size_t rows,
    size_t columns,
    size_t *out_count
) {
    if (out_count == NULL) {
        return false;
    }

    if (columns != 0 && rows > SIZE_MAX / columns) {
        return false;
    }

    size_t count = rows * columns;

    if (count > SIZE_MAX / sizeof(int)) {
        return false;
    }

    *out_count = count;
    return true;
}
```

The first guard protects element-count multiplication.

The second proves the element count can become a byte count for one C array object of `int`.

An allocation chapter would then use the checked count.

This chapter stops before allocation.

#### Explicit Stride Supports Views and Padding

Sometimes rows begin `stride` elements apart:

```txt
index = row * stride + column
columns <= stride
```

An explicit stride can describe:

- a rectangular subview inside a larger image
- padding at the end of each row
- a crop that shares another matrix's storage

The representation must distinguish:

```txt
columns = logical values per row
stride  = physical distance between row starts
```

Again, logical shape and physical capacity are not the same quantity.

### 41. Variable-Length Arrays Are Runtime-Sized, Not Resizable

At block scope, a non-constant bound can form a variable-length array:

```c
void process(size_t length) {
    if (length == 0) {
        return;
    }

    int values[length];

    /* Use values while this block is active. */
}
```

Important properties:

- the positive bound is evaluated when execution reaches the declaration
- the array's extent is then fixed
- changing `length` later does not resize `values`
- `sizeof values` is generally evaluated at runtime
- the array has automatic storage duration
- it ceases to exist when execution leaves its scope
- large bounds can exhaust limited automatic storage

A VLA is not a dynamic array data structure.

It has a runtime-determined fixed extent and no growth operation.

#### Portability

VLA support became optional for conforming C11 and C17 implementations.

Code can check implementation feature macros, provide another representation, or avoid VLA objects entirely.

This chapter's complete laboratory does not depend on VLAs.

#### A `const` Variable Is Not Necessarily a Compile-Time Bound in C

At block scope:

```c
const size_t count = 10;
int values[count];
```

`count` is not generally an integer constant expression in C.

This usually declares a VLA.

For a compile-time integral bound, use something such as:

```c
enum { COUNT = 10 };
int values[COUNT];
```

#### VLA Bounds Must Be Valid

When evaluated for an actual VLA object, the bound must be greater than zero.

Negative external input must be rejected before conversion to `size_t`.

This is dangerous:

```c
int user_count = read_count();
int values[(size_t)user_count];
```

A negative `user_count` can convert to a huge unsigned number.

Validation belongs before conversion and declaration.

### 42. Arrays of Structs and Structs of Arrays

Suppose every particle has:

```c
typedef struct {
    float x;
    float y;
    float velocity_x;
    float velocity_y;
} Particle;
```

An array of structs is:

```c
Particle particles[1000];
```

Layout:

```txt
[x y vx vy] [x y vx vy] [x y vx vy] ...
```

This keeps all fields of one particle together.

It is natural when each operation processes complete particles.

A struct of arrays is:

```c
typedef struct {
    float x[1000];
    float y[1000];
    float velocity_x[1000];
    float velocity_y[1000];
} ParticleFields;
```

Layout:

```txt
x:  [x x x x ...]
y:  [y y y y ...]
vx: [v v v v ...]
vy: [v v v v ...]
```

This can be useful when an operation scans only one or two fields.

#### Neither Layout Is Universally Better

The choice affects:

- which values are adjacent
- which data a traversal brings into cache
- how naturally one object can be copied or passed
- vectorization opportunities
- complexity of insertion and deletion across parallel fields

Both still use arrays.

The best layout follows the operations performed most often.

#### Element Assignment May Be Shallow

Consider:

```c
typedef struct {
    char *name;
    size_t score;
} Student;
```

This copies the pointer value:

```c
students[1] = students[0];
```

It does not allocate or copy the characters named by `name`.

If each element owns its `name`, naive array insertion, removal, copying, or destruction can create:

- two apparent owners of one allocation
- leaks
- double-free errors
- dangling pointers

An array provides element storage.

It does not define the resource-management semantics of the element type.

### 43. Contiguity Affects Real Machines Without Changing Big-O

Two traversals can both be `Theta(n)` and perform differently.

Sequential array traversal tends to have useful spatial locality:

```txt
after loading one region of memory,
nearby elements are likely already available or prefetched
```

An array also avoids a separate next-pointer load per element.

These practical properties often make array traversal efficient.

But avoid turning tendencies into unconditional promises.

Performance depends on:

- element size
- access order
- cache hierarchy
- compiler optimization
- vectorization
- memory bandwidth
- competing workloads
- the target machine

#### Access Pattern Matters

For a row-major matrix:

```c
for (size_t row = 0; row < rows; ++row) {
    for (size_t column = 0; column < columns; ++column) {
        use(matrix[row][column]);
    }
}
```

visits nearby elements in storage order.

Reversing the loop nesting:

```c
for (size_t column = 0; column < columns; ++column) {
    for (size_t row = 0; row < rows; ++row) {
        use(matrix[row][column]);
    }
}
```

jumps by a row stride between visits.

Both perform:

```txt
rows * columns
```

element operations and are:

```txt
Theta(rows * columns)
```

The first may still have much better locality.

Asymptotic analysis and machine-level performance answer different questions.

## Part V — Complexity and the Complete Laboratory

### 44. Complexity Summary

Let:

```txt
n = logical length
i = operation index
w = window width
d = value-domain size
q = number of queries or updates
```

| Operation | Time | Auxiliary space | Important condition |
|:---|:---|:---|:---|
| Index read or update | `Theta(1)` | `Theta(1)` | valid index |
| Traverse all elements | `Theta(n)` | `Theta(1)` | — |
| Linear search | best `Theta(1)`, worst `Theta(n)` | `Theta(1)` | unsorted allowed |
| Fixed-capacity append | `Theta(1)` | `Theta(1)` | fails when full |
| Insert at index `i` | `Theta(n - i + 1)` | `Theta(1)` | spare capacity |
| Stable remove at `i` | `Theta(n - i)` | `Theta(1)` | valid index |
| Unordered remove | `Theta(1)` | `Theta(1)` | order may change |
| Logical clear | `Theta(1)` | `Theta(1)` | bytes remain |
| Reverse | `Theta(n)` | `Theta(1)` | — |
| Rotate by reversals | trivial shift `Theta(1)`, otherwise `Theta(n)` | `Theta(1)` | — |
| Lower or upper bound | `Theta(log n)` | `Theta(1)` | sorted input |
| Sorted insertion | `Theta(n)` | `Theta(1)` | search plus shift |
| Stable compaction | `Theta(n)` | `Theta(1)` | — |
| Fixed-width window scan | `Theta(n)` | `Theta(1)` | valid width |
| Build prefix sums | `Theta(n)` | `Theta(n)` | sums fit |
| Prefix range query | `Theta(1)` | `Theta(1)` | prefix already built |
| `q` difference updates plus materialization | `Theta(q + n)` | `Theta(n)` | offline-style batching |
| Build frequency table | `Theta(n + d)` | `Theta(d)` | small bounded domain |
| Traverse rectangular matrix | `Theta(rows * columns)` | `Theta(1)` | valid shape |

#### There Is No Amortized Fixed-Array Append Here

Every successful append to this fixed-capacity structure performs constant work.

Every append when full fails in constant time.

There is no occasional resize whose expensive cost must be spread across cheaper operations.

Therefore amortized analysis is not needed for this operation.

In a dynamic array:

```txt
most appends may be cheap
some appends may allocate and copy many elements
```

That is where amortized append analysis becomes meaningful.

Keeping the two cases separate prevents a common misconception:

```txt
array append is always amortized O(1)
```

The true statement depends on which array representation and growth policy are being discussed.

### 45. Complete Fixed-Capacity Array Laboratory

The following program turns the chapter's invariants into one runnable C17 laboratory.

It deliberately has:

- caller-owned storage
- no allocation
- no resizing
- status values that distinguish ordinary failure cases
- failure-atomic operations
- tests for empty, partial, and full states
- tests for boundaries, duplicates, aliasing, and output preservation

Its status values distinguish:

```txt
INVALID       malformed descriptor or missing required output
OUT_OF_RANGE  index is not a legal element or insertion position
FULL          legal insertion position but no spare slot
EMPTY         pop requested from an empty logical sequence
NOT_FOUND     a valid search completed without a match
```

Every failure preserves the descriptor, every backing slot, and any required output object.

Validation therefore occurs before the first mutation.

For insertion, index validation precedes the full check:

```txt
index > length while full   -> OUT_OF_RANGE
index <= length while full  -> FULL
```

The representation is:

```txt
data      borrowed pointer to caller-owned storage
length    initialized logical prefix
capacity  writable physical slots
```

The structural predicate cannot prove:

- that `data` still points to a live object
- that the object really contains `capacity` elements
- that the first `length` slots were initialized by the caller

Those remain caller responsibilities.

The descriptor object itself must be separate from the backing array.

Required output objects must not overlap the descriptor or backing storage.

The implementation cannot portably discover every violation of that aliasing contract.

Save this as:

```txt
arrays_lab.c
```

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *data;
    size_t length;
    size_t capacity;
} IntArray;

typedef enum {
    INT_ARRAY_OK = 0,
    INT_ARRAY_INVALID,
    INT_ARRAY_OUT_OF_RANGE,
    INT_ARRAY_FULL,
    INT_ARRAY_EMPTY,
    INT_ARRAY_NOT_FOUND
} IntArrayStatus;

static bool int_array_is_valid(const IntArray *array) {
    if (array == NULL) {
        return false;
    }

    if (array->length > array->capacity) {
        return false;
    }

    if (array->capacity > SIZE_MAX / sizeof *array->data) {
        return false;
    }

    if (array->capacity == 0) {
        return array->data == NULL && array->length == 0;
    }

    return array->data != NULL;
}

static IntArrayStatus int_array_init(
    IntArray *array,
    int *storage,
    size_t length,
    size_t capacity
) {
    if (array == NULL ||
        length > capacity ||
        capacity > SIZE_MAX / sizeof *storage ||
        (capacity == 0 && storage != NULL) ||
        (capacity > 0 && storage == NULL)) {
        return INT_ARRAY_INVALID;
    }

    IntArray candidate = {
        .data = storage,
        .length = length,
        .capacity = capacity
    };

    *array = candidate;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_get(
    const IntArray *array,
    size_t index,
    int *out_value
) {
    if (!int_array_is_valid(array) || out_value == NULL) {
        return INT_ARRAY_INVALID;
    }

    if (index >= array->length) {
        return INT_ARRAY_OUT_OF_RANGE;
    }

    *out_value = array->data[index];
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_set(
    IntArray *array,
    size_t index,
    int value
) {
    if (!int_array_is_valid(array)) {
        return INT_ARRAY_INVALID;
    }

    if (index >= array->length) {
        return INT_ARRAY_OUT_OF_RANGE;
    }

    array->data[index] = value;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_append(
    IntArray *array,
    int value
) {
    if (!int_array_is_valid(array)) {
        return INT_ARRAY_INVALID;
    }

    if (array->length == array->capacity) {
        return INT_ARRAY_FULL;
    }

    array->data[array->length] = value;
    ++array->length;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_pop(
    IntArray *array,
    int *out_value
) {
    if (!int_array_is_valid(array) || out_value == NULL) {
        return INT_ARRAY_INVALID;
    }

    if (array->length == 0) {
        return INT_ARRAY_EMPTY;
    }

    int value = array->data[array->length - 1];
    --array->length;
    *out_value = value;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_insert(
    IntArray *array,
    size_t index,
    int value
) {
    if (!int_array_is_valid(array)) {
        return INT_ARRAY_INVALID;
    }

    if (index > array->length) {
        return INT_ARRAY_OUT_OF_RANGE;
    }

    if (array->length == array->capacity) {
        return INT_ARRAY_FULL;
    }

    for (size_t i = array->length; i > index; --i) {
        array->data[i] = array->data[i - 1];
    }

    array->data[index] = value;
    ++array->length;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_remove(
    IntArray *array,
    size_t index,
    int *out_value
) {
    if (!int_array_is_valid(array) || out_value == NULL) {
        return INT_ARRAY_INVALID;
    }

    if (index >= array->length) {
        return INT_ARRAY_OUT_OF_RANGE;
    }

    int value = array->data[index];

    for (size_t i = index; i < array->length - 1; ++i) {
        array->data[i] = array->data[i + 1];
    }

    --array->length;
    *out_value = value;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_remove_unordered(
    IntArray *array,
    size_t index,
    int *out_value
) {
    if (!int_array_is_valid(array) || out_value == NULL) {
        return INT_ARRAY_INVALID;
    }

    if (index >= array->length) {
        return INT_ARRAY_OUT_OF_RANGE;
    }

    int value = array->data[index];
    size_t last = array->length - 1;

    if (index != last) {
        array->data[index] = array->data[last];
    }

    --array->length;
    *out_value = value;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_find_first(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return INT_ARRAY_INVALID;
    }

    for (size_t i = 0; i < array->length; ++i) {
        if (array->data[i] == target) {
            *out_index = i;
            return INT_ARRAY_OK;
        }
    }

    return INT_ARRAY_NOT_FOUND;
}

static void reverse_range(
    int *data,
    size_t begin,
    size_t end
) {
    while (end - begin > 1) {
        --end;

        int temporary = data[begin];
        data[begin] = data[end];
        data[end] = temporary;

        ++begin;
    }
}

static IntArrayStatus int_array_reverse(IntArray *array) {
    if (!int_array_is_valid(array)) {
        return INT_ARRAY_INVALID;
    }

    if (array->length > 1) {
        reverse_range(array->data, 0, array->length);
    }

    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_rotate_left(
    IntArray *array,
    size_t shift
) {
    if (!int_array_is_valid(array)) {
        return INT_ARRAY_INVALID;
    }

    if (array->length < 2) {
        return INT_ARRAY_OK;
    }

    shift %= array->length;

    if (shift == 0) {
        return INT_ARRAY_OK;
    }

    reverse_range(array->data, 0, shift);
    reverse_range(array->data, shift, array->length);
    reverse_range(array->data, 0, array->length);
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_lower_bound(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return INT_ARRAY_INVALID;
    }

    size_t low = 0;
    size_t high = array->length;

    while (low < high) {
        size_t middle = low + (high - low) / 2;

        if (array->data[middle] < target) {
            low = middle + 1;
        } else {
            high = middle;
        }
    }

    *out_index = low;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_upper_bound(
    const IntArray *array,
    int target,
    size_t *out_index
) {
    if (!int_array_is_valid(array) || out_index == NULL) {
        return INT_ARRAY_INVALID;
    }

    size_t low = 0;
    size_t high = array->length;

    while (low < high) {
        size_t middle = low + (high - low) / 2;

        if (array->data[middle] <= target) {
            low = middle + 1;
        } else {
            high = middle;
        }
    }

    *out_index = low;
    return INT_ARRAY_OK;
}

static IntArrayStatus int_array_clear(IntArray *array) {
    if (!int_array_is_valid(array)) {
        return INT_ARRAY_INVALID;
    }

    array->length = 0;
    return INT_ARRAY_OK;
}

typedef struct {
    size_t assertions;
    size_t failures;
} TestContext;

static void test_check(
    TestContext *context,
    bool condition,
    const char *expression,
    const char *file,
    int line
) {
    ++context->assertions;

    if (!condition) {
        ++context->failures;
        fprintf(
            stderr,
            "%s:%d: CHECK failed: %s\n",
            file,
            line,
            expression
        );
    }
}

#define CHECK(context, expression) \
    test_check(                      \
        (context),                   \
        (expression),                \
        #expression,                 \
        __FILE__,                    \
        __LINE__                     \
    )

#define ARRAY_COUNT(array) \
    (sizeof(array) / sizeof((array)[0]))

static bool active_equals(
    const IntArray *array,
    const int *expected,
    size_t expected_length
) {
    if (!int_array_is_valid(array) ||
        array->length != expected_length) {
        return false;
    }

    if (expected_length > 0 && expected == NULL) {
        return false;
    }

    for (size_t i = 0; i < expected_length; ++i) {
        if (array->data[i] != expected[i]) {
            return false;
        }
    }

    return true;
}

static void test_init_and_aliasing(TestContext *context) {
    int storage[4] = {10, 20, 90, 91};
    IntArray array = {0};

    CHECK(
        context,
        int_array_init(&array, storage, 2, 4) == INT_ARRAY_OK
    );
    CHECK(context, int_array_is_valid(&array));
    CHECK(context, array.data == storage);
    CHECK(context, array.length == 2);
    CHECK(context, array.capacity == 4);

    CHECK(
        context,
        int_array_set(&array, 1, 25) == INT_ARRAY_OK
    );
    CHECK(context, storage[1] == 25);

    storage[0] = 11;
    int value = -1;
    CHECK(
        context,
        int_array_get(&array, 0, &value) == INT_ARRAY_OK
    );
    CHECK(context, value == 11);

    IntArray alias = array;
    CHECK(
        context,
        int_array_set(&alias, 0, 12) == INT_ARRAY_OK
    );
    CHECK(context, array.data[0] == 12);
    CHECK(context, alias.data == array.data);
}

static void test_init_failure_preserves_destination(
    TestContext *context
) {
    int storage[3] = {1, 2, 3};
    IntArray array = {
        .data = storage,
        .length = 2,
        .capacity = 3
    };

    CHECK(
        context,
        int_array_init(&array, storage, 4, 3) ==
            INT_ARRAY_INVALID
    );
    CHECK(context, array.data == storage);
    CHECK(context, array.length == 2);
    CHECK(context, array.capacity == 3);
    CHECK(context, storage[0] == 1);
    CHECK(context, storage[1] == 2);
    CHECK(context, storage[2] == 3);

    CHECK(
        context,
        int_array_init(&array, NULL, 0, 3) ==
            INT_ARRAY_INVALID
    );
    CHECK(context, array.data == storage);
    CHECK(context, array.length == 2);
    CHECK(context, array.capacity == 3);

    CHECK(
        context,
        int_array_init(&array, storage, 0, 0) ==
            INT_ARRAY_INVALID
    );
    CHECK(context, array.data == storage);
    CHECK(context, array.length == 2);
    CHECK(context, array.capacity == 3);
}

static void test_invalid_descriptors(TestContext *context) {
    int storage[2] = {10, 20};

    IntArray length_too_large = {
        .data = storage,
        .length = 2,
        .capacity = 1
    };
    CHECK(context, !int_array_is_valid(&length_too_large));

    IntArray missing_storage = {
        .data = NULL,
        .length = 0,
        .capacity = 2
    };
    CHECK(context, !int_array_is_valid(&missing_storage));

    IntArray noncanonical_zero = {
        .data = storage,
        .length = 0,
        .capacity = 0
    };
    CHECK(context, !int_array_is_valid(&noncanonical_zero));

    if (sizeof(int) > 1) {
        IntArray impossible_bytes = {
            .data = storage,
            .length = 0,
            .capacity = SIZE_MAX / sizeof(int) + 1
        };
        CHECK(context, !int_array_is_valid(&impossible_bytes));
    }
}

static void test_zero_capacity(TestContext *context) {
    IntArray array = {
        .data = NULL,
        .length = 0,
        .capacity = 0
    };

    CHECK(context, int_array_is_valid(&array));
    CHECK(
        context,
        int_array_clear(&array) == INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_reverse(&array) == INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_rotate_left(&array, SIZE_MAX) ==
            INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_append(&array, 10) == INT_ARRAY_FULL
    );

    int output = 777;
    CHECK(
        context,
        int_array_pop(&array, &output) == INT_ARRAY_EMPTY
    );
    CHECK(context, output == 777);
}

static void test_empty_with_capacity(TestContext *context) {
    int storage[2] = {90, 91};
    IntArray array = {
        .data = storage,
        .length = 0,
        .capacity = 2
    };

    CHECK(context, int_array_is_valid(&array));

    int output = 777;
    CHECK(
        context,
        int_array_get(&array, 0, &output) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, output == 777);
    CHECK(
        context,
        int_array_remove(&array, 0, &output) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, output == 777);

    CHECK(
        context,
        int_array_append(&array, 10) == INT_ARRAY_OK
    );
    CHECK(context, array.length == 1);
    CHECK(context, storage[0] == 10);
    CHECK(context, storage[1] == 91);
}

static void test_get_set_boundaries(TestContext *context) {
    int storage[5] = {10, 20, 30, 90, 91};
    IntArray array = {
        .data = storage,
        .length = 3,
        .capacity = 5
    };

    int output = -1;
    CHECK(
        context,
        int_array_get(&array, 0, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 10);
    CHECK(
        context,
        int_array_get(&array, 2, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 30);

    output = 777;
    CHECK(
        context,
        int_array_get(&array, 3, &output) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, output == 777);
    CHECK(
        context,
        int_array_get(&array, SIZE_MAX, &output) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, output == 777);

    CHECK(
        context,
        int_array_set(&array, 2, 35) == INT_ARRAY_OK
    );
    CHECK(context, storage[2] == 35);
    CHECK(
        context,
        int_array_set(&array, 3, 99) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, storage[3] == 90);
}

static void test_append_and_pop(TestContext *context) {
    int guarded[5] = {0x1357, 10, 80, 81, 0x2468};
    IntArray array = {
        .data = &guarded[1],
        .length = 1,
        .capacity = 3
    };

    CHECK(
        context,
        int_array_append(&array, 20) == INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_append(&array, 30) == INT_ARRAY_OK
    );

    int expected[] = {10, 20, 30};
    CHECK(
        context,
        active_equals(&array, expected, ARRAY_COUNT(expected))
    );
    CHECK(context, guarded[0] == 0x1357);
    CHECK(context, guarded[4] == 0x2468);

    CHECK(
        context,
        int_array_append(&array, 40) == INT_ARRAY_FULL
    );
    CHECK(
        context,
        active_equals(&array, expected, ARRAY_COUNT(expected))
    );
    CHECK(context, guarded[0] == 0x1357);
    CHECK(context, guarded[4] == 0x2468);

    int output = -1;
    CHECK(
        context,
        int_array_pop(&array, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 30);
    CHECK(context, array.length == 2);
    CHECK(
        context,
        int_array_pop(&array, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 20);
    CHECK(
        context,
        int_array_pop(&array, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 10);

    output = 777;
    CHECK(
        context,
        int_array_pop(&array, &output) == INT_ARRAY_EMPTY
    );
    CHECK(context, output == 777);
    CHECK(context, array.length == 0);
}

static void test_insert_boundaries(TestContext *context) {
    int storage[7] = {20, 30, 40, 90, 91, 92, 93};
    IntArray array = {
        .data = storage,
        .length = 3,
        .capacity = 7
    };

    CHECK(
        context,
        int_array_insert(&array, 0, 10) == INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_insert(&array, 2, 25) == INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_insert(&array, array.length, 50) ==
            INT_ARRAY_OK
    );

    int expected[] = {10, 20, 25, 30, 40, 50};
    CHECK(
        context,
        active_equals(&array, expected, ARRAY_COUNT(expected))
    );

    int snapshot[7];

    for (size_t i = 0; i < ARRAY_COUNT(snapshot); ++i) {
        snapshot[i] = storage[i];
    }

    CHECK(
        context,
        int_array_insert(&array, array.length + 1, 60) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, array.length == ARRAY_COUNT(expected));

    for (size_t i = 0; i < ARRAY_COUNT(snapshot); ++i) {
        CHECK(context, storage[i] == snapshot[i]);
    }
}

static void test_insert_full_is_atomic(TestContext *context) {
    int storage[3] = {10, 20, 30};
    IntArray array = {
        .data = storage,
        .length = 3,
        .capacity = 3
    };

    CHECK(
        context,
        int_array_insert(&array, 1, 99) == INT_ARRAY_FULL
    );
    CHECK(context, array.length == 3);
    CHECK(context, storage[0] == 10);
    CHECK(context, storage[1] == 20);
    CHECK(context, storage[2] == 30);

    CHECK(
        context,
        int_array_insert(&array, 4, 99) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, array.length == 3);
    CHECK(context, storage[0] == 10);
    CHECK(context, storage[1] == 20);
    CHECK(context, storage[2] == 30);
}

static void test_stable_remove_boundaries(TestContext *context) {
    int storage[5] = {10, 20, 30, 40, 50};
    IntArray array = {
        .data = storage,
        .length = 5,
        .capacity = 5
    };
    int output = -1;

    CHECK(
        context,
        int_array_remove(&array, 0, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 10);

    int after_front[] = {20, 30, 40, 50};
    CHECK(
        context,
        active_equals(
            &array,
            after_front,
            ARRAY_COUNT(after_front)
        )
    );

    CHECK(
        context,
        int_array_remove(&array, 1, &output) == INT_ARRAY_OK
    );
    CHECK(context, output == 30);

    int after_middle[] = {20, 40, 50};
    CHECK(
        context,
        active_equals(
            &array,
            after_middle,
            ARRAY_COUNT(after_middle)
        )
    );

    CHECK(
        context,
        int_array_remove(
            &array,
            array.length - 1,
            &output
        ) == INT_ARRAY_OK
    );
    CHECK(context, output == 50);

    int after_back[] = {20, 40};
    CHECK(
        context,
        active_equals(
            &array,
            after_back,
            ARRAY_COUNT(after_back)
        )
    );

    output = 777;
    CHECK(
        context,
        int_array_remove(&array, array.length, &output) ==
            INT_ARRAY_OUT_OF_RANGE
    );
    CHECK(context, output == 777);
    CHECK(
        context,
        active_equals(
            &array,
            after_back,
            ARRAY_COUNT(after_back)
        )
    );
}

static void test_unordered_remove(TestContext *context) {
    int storage[5] = {10, 20, 30, 40, 50};
    IntArray array = {
        .data = storage,
        .length = 5,
        .capacity = 5
    };
    int output = -1;

    CHECK(
        context,
        int_array_remove_unordered(&array, 1, &output) ==
            INT_ARRAY_OK
    );
    CHECK(context, output == 20);

    int expected[] = {10, 50, 30, 40};
    CHECK(
        context,
        active_equals(&array, expected, ARRAY_COUNT(expected))
    );

    CHECK(
        context,
        int_array_remove_unordered(
            &array,
            array.length - 1,
            &output
        ) == INT_ARRAY_OK
    );
    CHECK(context, output == 40);

    int after_last[] = {10, 50, 30};
    CHECK(
        context,
        active_equals(
            &array,
            after_last,
            ARRAY_COUNT(after_last)
        )
    );
}

static void test_find_first(TestContext *context) {
    int storage[4] = {4, 9, 2, 9};
    IntArray array = {
        .data = storage,
        .length = 4,
        .capacity = 4
    };

    size_t index = SIZE_MAX;
    CHECK(
        context,
        int_array_find_first(&array, 9, &index) ==
            INT_ARRAY_OK
    );
    CHECK(context, index == 1);

    index = 777;
    CHECK(
        context,
        int_array_find_first(&array, 8, &index) ==
            INT_ARRAY_NOT_FOUND
    );
    CHECK(context, index == 777);

    array.length = 0;
    CHECK(
        context,
        int_array_find_first(&array, 4, &index) ==
            INT_ARRAY_NOT_FOUND
    );
    CHECK(context, index == 777);
}

static void test_reverse_and_rotate(TestContext *context) {
    int odd_storage[5] = {1, 2, 3, 4, 5};
    IntArray odd = {
        .data = odd_storage,
        .length = 5,
        .capacity = 5
    };

    CHECK(
        context,
        int_array_reverse(&odd) == INT_ARRAY_OK
    );
    int reversed[] = {5, 4, 3, 2, 1};
    CHECK(
        context,
        active_equals(&odd, reversed, ARRAY_COUNT(reversed))
    );

    CHECK(
        context,
        int_array_reverse(&odd) == INT_ARRAY_OK
    );
    int original[] = {1, 2, 3, 4, 5};
    CHECK(
        context,
        active_equals(&odd, original, ARRAY_COUNT(original))
    );

    CHECK(
        context,
        int_array_rotate_left(&odd, 2) == INT_ARRAY_OK
    );
    int rotated[] = {3, 4, 5, 1, 2};
    CHECK(
        context,
        active_equals(&odd, rotated, ARRAY_COUNT(rotated))
    );

    CHECK(
        context,
        int_array_rotate_left(&odd, 5) == INT_ARRAY_OK
    );
    CHECK(
        context,
        active_equals(&odd, rotated, ARRAY_COUNT(rotated))
    );

    int even_storage[4] = {2, 4, 6, 8};
    IntArray even = {
        .data = even_storage,
        .length = 4,
        .capacity = 4
    };
    CHECK(
        context,
        int_array_reverse(&even) == INT_ARRAY_OK
    );
    int even_reversed[] = {8, 6, 4, 2};
    CHECK(
        context,
        active_equals(
            &even,
            even_reversed,
            ARRAY_COUNT(even_reversed)
        )
    );

    int one_storage[1] = {42};
    IntArray one = {
        .data = one_storage,
        .length = 1,
        .capacity = 1
    };
    CHECK(
        context,
        int_array_rotate_left(&one, SIZE_MAX) ==
            INT_ARRAY_OK
    );
    CHECK(context, one.data[0] == 42);
}

static void test_sorted_bounds(TestContext *context) {
    int storage[6] = {1, 2, 4, 4, 4, 9};
    IntArray array = {
        .data = storage,
        .length = 6,
        .capacity = 6
    };

    size_t lower = SIZE_MAX;
    size_t upper = SIZE_MAX;

    CHECK(
        context,
        int_array_lower_bound(&array, 4, &lower) ==
            INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_upper_bound(&array, 4, &upper) ==
            INT_ARRAY_OK
    );
    CHECK(context, lower == 2);
    CHECK(context, upper == 5);
    CHECK(context, upper - lower == 3);

    CHECK(
        context,
        int_array_lower_bound(&array, 0, &lower) ==
            INT_ARRAY_OK
    );
    CHECK(context, lower == 0);
    CHECK(
        context,
        int_array_upper_bound(&array, 100, &upper) ==
            INT_ARRAY_OK
    );
    CHECK(context, upper == array.length);

    CHECK(
        context,
        int_array_lower_bound(&array, 3, &lower) ==
            INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_upper_bound(&array, 3, &upper) ==
            INT_ARRAY_OK
    );
    CHECK(context, lower == 2);
    CHECK(context, upper == 2);

    array.length = 0;
    CHECK(
        context,
        int_array_lower_bound(&array, 4, &lower) ==
            INT_ARRAY_OK
    );
    CHECK(
        context,
        int_array_upper_bound(&array, 4, &upper) ==
            INT_ARRAY_OK
    );
    CHECK(context, lower == 0);
    CHECK(context, upper == 0);
}

static void test_clear_is_logical(TestContext *context) {
    int storage[4] = {10, 20, 30, 40};
    IntArray array = {
        .data = storage,
        .length = 3,
        .capacity = 4
    };

    CHECK(context, int_array_clear(&array) == INT_ARRAY_OK);
    CHECK(context, array.length == 0);
    CHECK(context, array.capacity == 4);
    CHECK(context, array.data == storage);
    CHECK(context, storage[0] == 10);
    CHECK(context, storage[1] == 20);
    CHECK(context, storage[2] == 30);
    CHECK(context, storage[3] == 40);
}

int main(void) {
    TestContext context = {0};

    test_init_and_aliasing(&context);
    test_init_failure_preserves_destination(&context);
    test_invalid_descriptors(&context);
    test_zero_capacity(&context);
    test_empty_with_capacity(&context);
    test_get_set_boundaries(&context);
    test_append_and_pop(&context);
    test_insert_boundaries(&context);
    test_insert_full_is_atomic(&context);
    test_stable_remove_boundaries(&context);
    test_unordered_remove(&context);
    test_find_first(&context);
    test_reverse_and_rotate(&context);
    test_sorted_bounds(&context);
    test_clear_is_logical(&context);

    printf(
        "Assertions: %zu, failures: %zu\n",
        context.assertions,
        context.failures
    );

    return context.failures == 0 ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

#### Compile Strictly

Debug build:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O0 -g3 \
   arrays_lab.c \
   -o arrays_lab

./arrays_lab
```

Sanitizer build with GCC or Clang:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O1 -g \
   -fsanitize=address,undefined \
   -fno-omit-frame-pointer \
   arrays_lab.c \
   -o arrays_lab_san

./arrays_lab_san
```

Release-like build:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -O2 -DNDEBUG \
   arrays_lab.c \
   -o arrays_lab_release

./arrays_lab_release
```

Static analysis with a recent GCC:

```sh
cc -std=c17 \
   -Wall -Wextra -Wpedantic -Wconversion -Werror \
   -fanalyzer \
   -c arrays_lab.c
```

#### What the Tests Prove

The tests provide evidence that:

- the view borrows rather than copies storage
- copying the view aliases the same storage
- rejected initialization preserves the previous destination
- zero-capacity operations avoid invalid storage access
- access boundaries distinguish `length` from `capacity`
- a full append and insertion preserve state
- insertion works at the front, middle, and end
- stable removal preserves relative order
- unordered removal intentionally changes order
- search returns the first duplicate
- failed outputs preserve their sentinels
- reverse handles odd and small arrays
- rotation handles large equivalent shifts
- bounds give a duplicate equal range
- clear changes logical state without overwriting storage

The tests do not prove all possible calls correct.

They also cannot detect a caller lying about:

- backing capacity
- initialization of active elements
- pointer lifetime
- forbidden output aliasing
- sortedness before a bounds search

Sanitizers can expose many physical out-of-bounds accesses.

They do not know that a physically in-bounds inactive slot is logically out of bounds.

Logical contracts still need tests and reasoning.

## Part VI — Review and Mastery

### 46. Edge-Case Matrix

Array algorithms often fail at transitions rather than ordinary middle states.

Use this matrix before calling an implementation complete.

| State or input | Questions to ask |
|:---|:---|
| Zero capacity | Is the representation canonical? Does code avoid pointer arithmetic and dereference? |
| Empty with positive capacity | Does access fail? Can append at index `0` succeed? Does reverse do nothing safely? |
| One logical element | Do first and last mean the same index? Can removal produce a valid empty state? |
| Partially full | Are inactive slots never read as logical values? |
| Exactly full | Do append and insertion fail without changing any slot or metadata? |
| Index `0` | Does front insertion/removal move the correct range? |
| Index `length - 1` | Does last access/removal avoid extra movement? |
| Index `length` | Is it accepted for insertion but rejected for access and removal? |
| Index `SIZE_MAX` | Does validation reject it before pointer arithmetic? |
| Duplicate values | Does search specify first, last, any, lower bound, or upper bound? |
| All values equal | Do bounds, compaction, and partition loops terminate? |
| Already reversed or sorted | Does the algorithm preserve its contract without assuming movement is required? |
| Rotation by zero | Is it a no-op? |
| Rotation by `length` or more | Is modulo used only after excluding zero length? |
| Negative external count | Is it rejected before conversion to `size_t`? |
| Huge element count | Are element and byte multiplications checked before evaluation? |
| Null pointer with length zero | Does the specific interface permit it, and does implementation avoid forming derived pointers? |
| Null pointer with positive length | Is it rejected? |
| Aliased view descriptors | Is shared mutation expected and documented? |
| Aliased source and destination ranges | Is overlap supported with `memmove`, or forbidden by contract? |
| Output inside backing storage | Is aliasing supported deliberately, or ruled out? |
| Expired backing storage | Does documentation make clear that structural validation cannot detect it? |
| Sorted-search input is unsorted | Is sortedness a stated caller precondition? |
| Matrix with zero dimension | What logical representation is allowed, and does code avoid invalid VLA bounds? |
| Matrix row/column at boundary | Are both dimensions checked before flattening? |

#### Empty and Zero-Capacity Are Not Synonyms

These states are both logically empty:

```txt
data = NULL,       length = 0, capacity = 0
data = valid ptr,  length = 0, capacity = 8
```

The first cannot accept an append.

The second can.

Logical contents alone do not describe every possible operation.

Capacity is part of mutable-sequence state.

#### Full Is Not an Invalid Structure

This is valid:

```txt
length == capacity
```

It means every slot is active.

Append fails because no spare slot exists, not because the structure invariant is broken.

Distinguishing valid-but-full from invalid makes APIs and tests clearer.

#### A Valid-Looking Descriptor Can Still Lie

This structure passes a shallow field check:

```c
IntArray claimed = {
    .data = some_pointer,
    .length = 4,
    .capacity = 8
};
```

The predicate cannot discover whether:

- `some_pointer` is dangling
- only two elements actually exist
- the active slots are initialized
- the storage is writable

C interfaces depend on contracts that extend beyond field arithmetic.

Validation catches what representation permits it to catch.

It is not a pointer-validity oracle.

### 47. Common Bugs and Why They Fail

#### Bug 1: Using `<=` for Element Traversal

Wrong:

```c
for (size_t i = 0; i <= length; ++i) {
    use(data[i]);
}
```

At `i == length`, the loop dereferences the one-past position.

Correct:

```c
for (size_t i = 0; i < length; ++i) {
    use(data[i]);
}
```

#### Bug 2: Starting a Reverse Loop at `length - 1`

Wrong for an empty array:

```c
size_t i = length - 1;
```

Unsigned underflow produces a huge value.

Start at the one-past count and subtract only after proving it is positive:

```c
for (size_t i = length; i > 0; --i) {
    use(data[i - 1]);
}
```

#### Bug 3: Shifting an Insertion Forward

Wrong:

```c
for (size_t i = index; i < length; ++i) {
    data[i + 1] = data[i];
}
```

The first assignment overwrites a future source.

Rightward overlapping movement must proceed backward.

#### Bug 4: Shifting a Removal Backward

Removal moves each source toward a lower index.

Copying high-to-low can overwrite a source before it is read.

Leftward overlapping movement must proceed forward.

#### Bug 5: Confusing Capacity With Length

Wrong:

```c
for (size_t i = 0; i < array->capacity; ++i) {
    sum += array->data[i];
}
```

Inactive slots may be indeterminate or stale.

Logical algorithms traverse `length`.

Capacity is used to validate writes that may extend the logical prefix.

#### Bug 6: Publishing Length Before the Value

Wrong pattern:

```c
++array->length;
array->data[array->length - 1] = value;
```

In more complex operations, a failure between those actions would expose an unwritten logical slot.

Validate, write the new value, then commit the new length.

#### Bug 7: Recovering Length Inside a Parameter

Wrong:

```c
void print_values(int values[]) {
    size_t length = sizeof values / sizeof values[0];
}
```

Inside the function, `values` is a pointer parameter.

Pass length separately.

#### Bug 8: Calling an Array a Const Pointer

An array is not a pointer object and stores no reseatable address value.

The array-to-pointer conversion produces a pointer in most expressions.

That conversion does not change the array object's type or identity.

#### Bug 9: Returning a View Into Automatic Storage

Wrong:

```c
int *make_values(void) {
    int values[8];
    return values;
}
```

The array lifetime ends at function return.

The pointer dangles immediately.

#### Bug 10: Reading an Uninitialized Automatic Element

Wrong:

```c
int values[8];
printf("%d\n", values[0]);
```

The element has an indeterminate value.

Reading it this way has undefined behavior.

Do not describe this as merely receiving “random garbage.”

#### Bug 11: Using `memset` to Create Integer Ones

Wrong:

```c
memset(values, 1, sizeof values);
```

`memset` fills every byte with the byte value `1`.

It does not assign the integer value `1` to each element.

Use an element loop:

```c
for (size_t i = 0; i < ARRAY_COUNT(values); ++i) {
    values[i] = 1;
}
```

#### Bug 12: Using `memcpy` for an Overlapping Shift

Insertion and stable removal use overlapping ranges.

`memcpy` requires non-overlap.

Use a directionally correct element loop or `memmove`.

#### Bug 13: Assuming Zero Bytes Excuse Invalid Pointers

Code often computes a source or destination pointer before discovering the count is zero.

Handle an empty case before:

- null pointer arithmetic
- underflowing `length - 1`
- constructing an invalid subscript
- calling a memory function with a questionable operand

Zero work should produce a naturally safe path.

#### Bug 14: Checking Overflow After Multiplication

Wrong:

```c
size_t bytes = count * sizeof *data;

if (bytes > SIZE_MAX) {
    return false;
}
```

`bytes` can never be greater than `SIZE_MAX`, and overflow has already wrapped.

Check before multiplication:

```c
if (count > SIZE_MAX / sizeof *data) {
    return false;
}

size_t bytes = count * sizeof *data;
```

#### Bug 15: Converting a Negative Count Too Early

Wrong:

```c
int input = read_count();
size_t length = (size_t)input;
```

A negative value converts to a large unsigned value.

Validate in the signed domain first.

#### Bug 16: Overflowing the Binary-Search Midpoint

Avoid:

```c
size_t middle = (low + high) / 2;
```

Use:

```c
size_t middle = low + (high - low) / 2;
```

#### Bug 17: Overflowing a Sort Comparator

This comparator is unsafe:

```c
return left_value - right_value;
```

The subtraction can overflow.

A safe three-way result is:

```c
return (left_value > right_value) -
       (left_value < right_value);
```

General sorting algorithms are outside this chapter, but sorted-array invariants depend on correct comparison semantics.

#### Bug 18: Treating `int **` as a Rectangular Array

The pointer depth looks suggestive but the representations differ.

A rectangular `int[R][C]` has row objects embedded contiguously.

An `int **` points to pointer objects.

A cast cannot manufacture the missing row-pointer table or correct stride.

#### Bug 19: Using a VLA as an Unbounded Convenience

Runtime size does not mean unlimited safe size.

A huge VLA can exhaust automatic storage, and a zero bound is invalid for an actual VLA object.

Use it only with a validated, deliberately bounded size and a portability policy.

#### Bug 20: Assuming a Copied View Is an Independent Copy

```c
IntArray second = first;
```

copies:

```txt
pointer
length
capacity
```

Both descriptors point at the same element storage.

They can even develop different `length` fields while still sharing the bytes, which may make reasoning especially confusing.

Document whether descriptor copying is allowed and what aliasing means.

### 48. Common Misconceptions

#### “An Array Is a Pointer”

No.

An array is an object containing a fixed sequence of element objects.

A pointer is a scalar object whose value can identify another object.

Array expressions often convert to pointers, which is why the two are easily confused.

#### “The Array Stores Its Length at Runtime”

An ordinary C array type includes an extent where that type is known.

After conversion to a pointer, the pointer value carries no portable length metadata.

An application must preserve length separately when needed.

#### “Constant-Time Access Means Any Array Operation Is Constant”

Index-to-address calculation is constant time.

Search, traversal, insertion, stable deletion, and reversal have different work.

Analyze the requested operation.

#### “One Past Is the Last Element”

For length `n`:

```txt
last element index = n - 1, when n > 0
one-past index     = n
```

The one-past pointer is useful as a boundary and must not be dereferenced.

#### “If Physical Storage Exists, It Is Safe to Read”

The backing array may contain a slot at index `i`.

If `i >= logical length`, that slot is not an element of the abstract sequence.

It may also be uninitialized.

Physical bounds and logical bounds both matter.

#### “A Full Array Is Broken”

Full is a valid state.

It simply cannot accept an operation that needs another slot.

#### “Deleting Requires Zeroing the Old Last Slot”

Stable deletion reduces logical length after shifting.

The retired slot is inactive.

Zeroing it is a separate policy, not a requirement for sequence correctness.

#### “Insertion at the End Is Linear”

Insertion in general has a linear worst case.

At `index == length`, no existing element moves, so fixed-capacity insertion is constant time.

State both the parameterized and worst-case costs.

#### “Binary Search Makes Sorted Insertion Logarithmic”

It finds a position logarithmically.

Opening a contiguous gap remains linear in the worst case.

#### “Any Binary-Search Match Is Good Enough”

Only if the contract says any duplicate is acceptable.

Clients may require first equal, last equal, lower bound, upper bound, or an equal range.

#### “`memmove` Is Always Constant Time Because It Is a Library Call”

A function call can hide a loop without removing its work.

Moving `m` bytes is `Theta(m)`.

#### “A VLA Is a Dynamic Array”

A VLA chooses its fixed extent at runtime.

A dynamic array owns replaceable storage and may grow or shrink.

They solve different problems.

#### “Two Dimensions Mean `int **`”

A two-dimensional array is an array of row arrays.

Its converted pointer points to a row type, not to an `int *` object.

#### “Big-O Predicts Exact Speed”

Big-O describes growth under an abstract cost model.

Contiguity, element size, cache behavior, compiler decisions, and hardware still affect actual time.

#### “Validation Can Prove a Pointer Is Safe”

Field checks can reject null pointers, impossible lengths, and byte-count overflow.

They cannot prove lifetime, actual allocation extent, initialization, or ownership.

Those facts come from construction and caller discipline.

### 49. A Repeatable Method for Array Problems

When facing an array operation, reason in this order.

#### Step 1: Name the Representation

Write down:

```txt
element type
physical capacity
logical length
ownership
storage lifetime
order invariant, if any
```

Do not start with a loop while those facts are ambiguous.

#### Step 2: Define the Legal Range

Choose a half-open range:

```txt
[0, length)
[begin, end)
```

State whether a position may equal `length`.

Insertion positions and element indices are not the same set.

#### Step 3: State Preconditions and Outcomes

Ask:

- Can the pointer be null for an empty range?
- Must storage be writable?
- Is spare capacity required?
- Must input be sorted?
- May ranges overlap?
- What happens with duplicates?
- What remains unchanged on failure?

#### Step 4: Draw Before and After States

For a mutation, mark:

```txt
unchanged prefix
moved region
new or removed position
unchanged suffix
new logical length
```

This usually reveals shift direction.

#### Step 5: Choose a Loop Invariant

Examples:

```txt
insertion:
the suffix already copied to the right is correct

stable compaction:
data[0, write) is the retained subsequence of old[0, read)

binary lower bound:
indices before low are too small
indices at or after high are large enough

partition:
left region is classified one way
right region is classified the other way
middle remains unknown

sliding window:
current equals the sum of the current window
```

The invariant explains why partial progress remains correct.

#### Step 6: Prove Every Index Before Dereference

Immediately before:

```c
data[index]
```

identify the fact establishing:

```txt
index < valid bound
```

Immediately before:

```c
length - 1
```

identify the fact establishing:

```txt
length > 0
```

#### Step 7: Separate Element Counts From Byte Counts

Reason first in elements:

```txt
move count elements
```

Then convert safely:

```txt
count * sizeof element
```

with a representation invariant or explicit overflow guard.

#### Step 8: Dry Run the Transitions

At minimum use:

- empty
- one element
- first position
- middle position
- last position
- full capacity
- duplicate values where relevant

Record every changed index.

#### Step 9: Count the Governing Operation

Do not count syntax.

Count:

- inspected elements
- moved elements
- comparisons
- window-edge movements
- constructed prefix entries

Express a more precise cost before taking a worst case:

```txt
insertion moves n - i elements
```

Then:

```txt
worst case Theta(n)
```

#### Step 10: Test the Contract, Not Only the Happy Path

Check:

- returned status
- logical sequence
- length and capacity
- inactive slots when failure preservation promises them unchanged
- output sentinels
- boundary guards
- later usability after failure

The strongest array reasoning joins representation, proof, dry run, complexity, and executable observation.

### 50. Challenging Practice Questions

These questions are designed to require contracts and reasoning, not only code.

#### Core Mechanics and Proofs

Questions 1 through 13 use only the array representation, movement, and search material developed in Parts I and II.

##### Question 1: Four Meanings of Size

For:

```c
int storage[12];
IntArray values = {storage, 5, 12};
```

identify:

- element size
- declared extent
- physical byte size
- logical length
- capacity
- logical byte count

State which quantities remain recoverable inside a function receiving only `int *data`.

##### Question 2: Address Types

Given:

```c
int values[6];
```

state the types and pointer-arithmetic steps of:

```c
values
&values[0]
&values
values + 1
&values + 1
```

Explain why equal printed starting addresses do not imply equal types.

##### Question 3: Array Parameter Audit

Diagnose:

```c
static size_t count(int values[20]) {
    return sizeof values / sizeof values[0];
}
```

Rewrite the interface.

Then explain what `int values[static 20]` would and would not guarantee.

##### Question 4: Reverse Loop Proof

Prove that:

```c
for (size_t i = length; i > 0; --i) {
    visit(data[i - 1]);
}
```

visits every valid index exactly once for:

- length zero
- length one
- arbitrary positive length

Explain precisely why the common `i >= 0` form fails.

##### Question 5: Failure-Atomic Append

Specify an append contract for a fixed-capacity array.

Design tests proving:

- success into the last free slot
- failure when full
- preservation of all metadata and physical slots on failure
- boundary guards remain unchanged

##### Question 6: Insertion Movement Count

For logical length `n`, derive the exact number of element assignments required to insert at index `i`.

Cover:

- `i = 0`
- `i = n / 2`
- `i = n`

Separate shift assignments from the new-value assignment.

##### Question 7: Find the Shift Bug

This insertion corrupts data:

```c
for (size_t i = index; i < length; ++i) {
    data[i + 1] = data[i];
}
```

Dry-run it on:

```txt
[10, 20, 30, 40]
```

inserting `15` at index `1`.

Show the first point at which information is lost and repair the loop.

##### Question 8: Stable Versus Unordered Removal

Implement both policies.

For each, state:

- postcondition
- exact moved-element count
- best and worst cases
- a client for which that policy is invalid

##### Question 9: Aliasing Output

Consider:

```c
int_array_remove(&array, 1, &array.data[3]);
```

Analyze why an output pointer into the moved range complicates failure and success semantics.

Choose one:

- support all overlap deliberately
- support a restricted set of aliases
- forbid output overlap

Write the exact contract and tests.

##### Question 10: Overlapping Bulk Movement

Rewrite insertion and stable removal with `memmove`.

For each call, derive:

- source half-open range
- destination half-open range
- element count
- byte count
- proof that endpoints remain within the backing array or one past it
- proof that byte multiplication cannot overflow

##### Question 11: Duplicate Bounds

For:

```txt
[1, 2, 2, 2, 5, 8]
```

dry-run lower and upper bound for targets:

```txt
0, 2, 3, 9
```

Derive membership and occurrence count from the two results.

##### Question 12: Binary-Search Invariant

Prove the lower-bound loop.

Your argument must establish:

- initialization of the invariant
- preservation in both branches
- strict progress
- termination
- correctness when the range becomes empty
- safety of midpoint arithmetic

##### Question 13: Sorted Insertion Complexity

An engineer claims:

> We use binary search, so insertion into our sorted array is `O(log n)`.

Give the tight worst-case bound.

Separate comparison cost from movement cost and construct an input that reaches it.

#### Applied Array Patterns

Questions 14 through 20 build on Part III's compaction, partition, window, prefix, difference, and frequency patterns.

##### Question 14: Stable Compaction

Write a general retain operation using a predicate:

```c
bool retain(int value, void *context);
```

State an aliasing rule for `context`.

Prove stability and show why `write <= read` prevents destroying unread input.

##### Question 15: Move Zeroes

Transform:

```txt
[0, 4, 0, 2, 3, 0]
```

into:

```txt
[4, 2, 3, 0, 0, 0]
```

while preserving nonzero order.

Do it in:

```txt
Theta(n) time
Theta(1) auxiliary space
```

Distinguish logical compaction from the requirement to write zeros into the suffix.

##### Question 16: Partition Contract

Adapt the two-pointer partition to place values less than a pivot before values not less than it.

State whether the result is stable.

Give a loop invariant and prove termination for:

- all values below the pivot
- no values below the pivot
- all values equal to the pivot

##### Question 17: Variable Sliding Window

For an array of nonnegative integers, find the shortest nonempty subarray whose sum is at least a target.

Derive a linear algorithm.

Explain exactly which monotonic fact fails when negative values are permitted, and provide a counterexample to naive shrinking.

##### Question 18: Prefix-Sum Overflow

Design a prefix-sum API for arbitrary `int` input.

Choose one policy:

- reject any sum outside `intmax_t`
- use a documented input bound that proves sums fit
- use a wider implementation-specific type

Write the checks or mathematical precondition.

Explain why simply changing the output element type does not solve unbounded accumulation.

##### Question 19: Difference Array

For length `8`, apply:

```txt
add 3 to [0, 4)
add -2 to [2, 7)
add 5 to [6, 8)
```

Show every difference entry and reconstruct the final values.

State the extra-space and time costs for `q` updates.

##### Question 20: Frequency Domain Mapping

Values are promised to lie in:

```txt
[-50, 75]
```

Design a frequency array.

Derive:

- domain size without signed overflow
- mapping from value to index
- validation before conversion to `size_t`
- inverse mapping

Then explain why the approach is unsuitable for the complete `int` domain.

#### Systems, Ownership, and Testing

Questions 21 through 28 combine array mechanics with C representation, lifetime, overflow, ownership, and the testing discipline from the preceding chapter.

##### Question 21: Matrix Type and Layout

For:

```c
int matrix[3][5];
```

state the types of:

```c
matrix
matrix[0]
&matrix[0]
&matrix[0][0]
```

Derive:

```c
sizeof matrix
sizeof matrix[0]
```

without assuming any particular byte size for `int`.

##### Question 22: Flat Matrix Safety

Design:

```c
bool matrix_get(
    const int *data,
    size_t storage_count,
    size_t rows,
    size_t columns,
    size_t row,
    size_t column,
    int *out_value
);
```

Validate:

- shape multiplication
- claimed storage
- row and column
- flattened index
- null pointers
- output preservation

State which facts still cannot be verified.

##### Question 23: Strided Submatrix View

A flat image has:

```txt
width = 1920
height = 1080
stride = 2048 pixels
```

Design a non-owning crop view.

Derive address calculation for crop coordinate `(r, c)`.

Check every multiplication and addition for overflow.

State the lifetime and aliasing rules between the crop and source image.

##### Question 24: VLA Portability Review

Review:

```c
void work(int count) {
    const size_t n = (size_t)count;
    int values[n];
    /* ... */
}
```

Identify:

- negative-input conversion
- zero-bound behavior
- whether this is a VLA in C
- automatic-storage risk
- optional C17 support
- lifetime

Rewrite it with a deliberate maximum bound and a fallback for implementations without VLAs.

##### Question 25: Arrays of Owning Elements

An array contains:

```c
typedef struct {
    char *text;
} Message;
```

Every element owns its string.

Specify safe contracts for:

- element insertion
- stable movement
- element removal
- array cloning
- clear

Distinguish representation relocation, ownership transfer, shallow copy, and deep copy.

##### Question 26: Merge Two Sorted Ranges

Given two sorted arrays of lengths `left_length` and `right_length`, merge them into caller-provided output storage.

Your solution must:

- validate output capacity without overflowing the sum
- define duplicate ordering
- specify whether output may overlap either input
- run in `Theta(left_length + right_length)` time
- preserve output on detectable validation failure

Dry-run unequal lengths and exhausted-left/exhausted-right cases.

##### Question 27: Rotate Tradeoffs

Implement left rotation using:

1. repeated one-position shifts
2. a temporary array
3. three reversals

Compare:

- time as a function of `n` and `k`
- auxiliary space
- overwrite risks
- empty-input handling
- behavior when `k >= n`

##### Question 28: Model-Based Fixed-Array Test

Create a simple reference model and generate deterministic sequences of:

- append
- insert
- stable remove
- unordered remove
- set
- clear

After every step compare:

- returned status
- logical length
- every logical value
- capacity
- boundary guards

Print the seed and operation index on failure.

Explain why deterministic direct tests should remain even after this model test exists.

### 51. Final Check

Before moving on, I should be able to explain:

- why an array is one object containing contiguous element objects
- how element type and extent determine layout
- why a standard ordinary array cannot have zero extent
- how automatic, static, and partial initialization differ
- why `a[i]` follows from pointer arithmetic
- why valid logical indices form `[0, length)`
- why a one-past pointer may be formed but not dereferenced
- why arrays and pointers are different even though conversion is common
- what happens to array parameters at function boundaries
- when `sizeof array / sizeof array[0]` works and when it fails
- why logical length and physical capacity must be separate
- why a non-owning descriptor cannot prove storage lifetime
- why direct access is constant time but search is not
- why rightward overlapping movement runs backward
- why leftward overlapping movement runs forward
- why insertion accepts `index == length` but access does not
- how fixed-capacity append differs from dynamic-array append
- why logical clear need not erase bytes
- when unordered removal is valid
- why `memmove` is required for overlapping bulk ranges
- why byte counts require overflow reasoning
- how lower and upper bounds handle duplicates precisely
- why sorted insertion remains linear
- how reversal and rotation avoid unsigned underflow
- what stable compaction's read/write invariant means
- why the two-pointer partition is usually unstable
- what reused work makes a sliding window linear
- which monotonic condition a variable window needs
- how an `n + 1` prefix array answers half-open range sums
- when difference and frequency arrays are appropriate
- why a two-dimensional array decays to a pointer to a row
- why `int **` is not a rectangular array
- how flat matrix indexing can overflow
- why a VLA is runtime-sized but not resizable
- how arrays of structs and structs of arrays serve different access patterns
- why copying an element can still be shallow
- how contiguity affects locality without changing asymptotic notation
- which edge states every array operation must test
- how to derive a loop from a representation invariant and postcondition

If I cannot identify the active range, prove every subscript valid, and state what happens at zero length and full capacity, I do not yet understand the array operation.
