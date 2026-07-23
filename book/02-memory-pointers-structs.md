# Memory, Pointers, and Structs in C

Data structures are arrangements of memory.

Before I build arrays, linked lists, stacks, queues, trees, or hash tables, I need a precise model of what the C program is manipulating.

A pointer is not merely “a variable that stores an address.”

A useful pointer must refer to the right kind of object, during that object's lifetime, with the required alignment, and within the bounds where the intended operation is valid.

A dynamically allocated block is not merely “heap memory.”

Someone must own it, preserve the only usable pointer to it, decide when its lifetime ends, and prevent every later access.

A `struct` is not merely “a group of variables.”

It is a new object type with a layout, alignment, invariants, copy behavior, and possibly ownership responsibilities.

The goal of this chapter is to make those mechanics explicit.

## 1. Begin With Objects, Not Boxes

It is common to draw a variable as a labeled box:

```txt
count
+----+
| 17 |
+----+
```

The drawing is useful, but it hides important facts.

In C, an object is a region of data storage with:

- a type
- a size
- an alignment requirement
- a value or representation
- a lifetime
- an address while it exists

Consider:

```c
int count = 17;
```

The declaration asks for an object named `count` whose type is `int`.

The implementation reserves `sizeof(int)` bytes at a suitably aligned location and stores a representation of the value `17` there.

The C standard does not require `int` to be exactly four bytes, although that is common.

I can ask the implementation:

```c
#include <stdio.h>

int main(void) {
    int count = 17;

    printf("value: %d\n", count);
    printf("size: %zu bytes\n", sizeof count);
    printf("address: %p\n", (void *)&count);

    return 0;
}
```

`sizeof` returns a value of type `size_t`.

`%zu` is the matching `printf` conversion.

`%p` expects a `void *`, so I explicitly convert `&count` for printing.

The printed address may change between executions.

The important point is not the numeric address.

The important point is that the object occupies storage, and expressions can either use its value or refer to its location.

## 2. Bytes, Types, and Interpretation

Memory can be viewed as a sequence of addressable bytes:

```txt
lower address                                  higher address
     |                                              |
     v                                              v
+--------+--------+--------+--------+--------+--------+
| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 | byte 5 |
+--------+--------+--------+--------+--------+--------+
```

The bytes alone do not tell me the full meaning of an object.

The type determines matters such as:

- how many bytes belong to the object
- how those bytes are interpreted
- which operations are permitted
- what alignment the address must satisfy
- how pointer arithmetic advances

Suppose `int` occupies four bytes on a particular implementation.

Then this:

```c
int values[3] = {10, 20, 30};
```

may occupy twelve contiguous bytes.

Conceptually:

```txt
values[0]       values[1]       values[2]
+-----------+   +-----------+   +-----------+
| 4 bytes   |   | 4 bytes   |   | 4 bytes   |
+-----------+   +-----------+   +-----------+
^
values
```

The exact byte representation and byte order are implementation details.

The structural guarantee is more important here:

> Array elements are contiguous, have the same type, and appear in index order.

That guarantee is what makes constant-time indexing and pointer arithmetic possible.

### Inspecting an Object Representation

C permits the bytes of an object to be inspected through a character type.

```c
#include <stddef.h>
#include <stdio.h>

void print_bytes(const void *object, size_t size) {
    const unsigned char *bytes = object;

    for (size_t i = 0; i < size; ++i) {
        printf("%02X%s", bytes[i], i + 1 == size ? "\n" : " ");
    }
}

int main(void) {
    unsigned int value = 0x12345678U;

    print_bytes(&value, sizeof value);
    return 0;
}
```

The output may reveal the machine's byte order, but I should not write portable code that assumes one particular result unless the format explicitly defines it.

Raw bytes are representation.

The declared type gives those bytes their ordinary program meaning.

## 3. Declarations Must Be Read Precisely

These declarations are different:

```c
int value;
int *pointer;
int values[10];
int *pointers[10];
int (*pointer_to_array)[10];
```

They mean:

- `value` is an `int`
- `pointer` is a pointer to `int`
- `values` is an array of ten `int`
- `pointers` is an array of ten pointers to `int`
- `pointer_to_array` is a pointer to an array of ten `int`

Parentheses matter.

Without them:

```c
int *pointers[10];
```

the `[]` binds first, so `pointers` is an array.

With them:

```c
int (*pointer_to_array)[10];
```

the name is first bound to `*`, so it is a pointer.

A practical reading method is:

1. Start at the declared name.
2. Look right for array brackets or function parentheses.
3. Look left for pointer stars and qualifiers.
4. Continue outward until the base type is reached.

I should not proceed while a declaration is still visually mysterious.

Data-structure code uses declarations to state the shape of the memory being manipulated.

## 4. Storage Duration and Lifetime

An object's lifetime is the period during which that object exists.

Using storage outside the object's lifetime is invalid even if the numeric address still appears unchanged.

For the work in this book, three storage patterns matter most.

### Automatic Storage Duration

```c
void example(void) {
    int local = 42;
}
```

`local` begins its lifetime when execution enters its block and its declaration is reached.

Its lifetime ends when execution leaves the block.

Automatic objects are commonly implemented with a call stack, but the C language specifies behavior rather than requiring a particular physical stack.

Returning a pointer to such an object is wrong:

```c
int *bad_pointer(void) {
    int local = 42;
    return &local;  /* Wrong: local dies when the function returns. */
}
```

The pointer value may still resemble an address.

It no longer points to a live `int` object.

Dereferencing it has undefined behavior.

### Static Storage Duration

```c
static int calls;

void record_call(void) {
    ++calls;
}
```

`calls` exists for the entire execution of the program.

Objects declared at file scope have static storage duration.

A block-scope object declared with `static` also has static storage duration, although its name remains visible only in that block.

Static lifetime can make a returned pointer valid:

```c
int *shared_counter(void) {
    static int counter;
    return &counter;
}
```

But the function now exposes shared mutable state.

Every caller receives the same object.

Lifetime safety does not automatically imply good interface design.

### Allocated Storage Duration

```c
int *value = malloc(sizeof *value);
```

If the allocation succeeds, the allocated object exists until the storage is released with `free` or successfully resized by `realloc`.

The lifetime is controlled manually.

This flexibility is necessary for data structures whose size or lifetime is determined at runtime.

It also creates responsibilities:

- check whether allocation succeeded
- retain a pointer that can release the block
- define who owns the block
- release it exactly once
- stop all access after release

The informal terms “stack” and “heap” are useful implementation shorthand.

The language-level questions—type, lifetime, bounds, and ownership—are more reliable.

## 5. A Pointer Is a Typed Reference to Storage

Consider:

```c
int value = 25;
int *pointer = &value;
```

`&value` produces the address of `value`.

`pointer` stores a pointer to `int`.

Conceptually:

```txt
pointer                              value
+------------------+                 +------+
| address of value | --------------> |  25  |
+------------------+                 +------+
```

The expression:

```c
*pointer
```

dereferences the pointer and designates the pointed-to `int`.

Therefore:

```c
*pointer = 90;
```

changes `value`.

```c
#include <stdio.h>

int main(void) {
    int value = 25;
    int *pointer = &value;

    *pointer = 90;

    printf("%d\n", value);  /* 90 */
    return 0;
}
```

The star has two related uses:

```c
int *pointer;  /* Declaration: pointer has type pointer-to-int. */
*pointer = 90; /* Expression: access the pointed-to int. */
```

The declaration introduces a pointer type.

The expression follows a pointer to the object it designates.

## 6. The Pointer Validity Checklist

Before dereferencing a pointer, I should be able to justify all of the following:

1. The pointer is not null.
2. It points to a live object.
3. The address satisfies the required alignment.
4. The access uses a permitted type.
5. The access remains within the object's valid bounds.
6. The intended operation is allowed by `const` qualification.

This is a stronger model than:

> The pointer contains an address, so it should work.

An arbitrary integer converted to a pointer may contain a numeric address and still fail every useful validity condition.

An old pointer to released storage may be non-null and still be invalid.

A pointer one element past an array is valid for comparison and subtraction within that array, but not for dereference.

The checklist is the core safety invariant behind pointer-based code.

## 7. Null Pointers Represent “No Object”

A null pointer does not point to an object.

```c
int *pointer = NULL;
```

`NULL` is provided by headers such as `<stddef.h>`.

Checking a pointer is straightforward:

```c
if (pointer != NULL) {
    printf("%d\n", *pointer);
}
```

This is also commonly written:

```c
if (pointer) {
    printf("%d\n", *pointer);
}
```

Dereferencing a null pointer has undefined behavior.

This is wrong:

```c
int *pointer = NULL;
*pointer = 10;
```

Null is useful only when the interface assigns it a clear meaning.

Examples include:

- an optional link is absent
- a search did not find an object
- allocation failed
- an owning structure currently owns no allocation

I should not use null as a substitute for documenting the object's state.

For example, an empty dynamic array may have:

```txt
data = NULL
size = 0
capacity = 0
```

The relationship among all three fields is the invariant.

## 8. C Passes Arguments by Value

C does not pass variables themselves into functions.

It evaluates each argument and copies the resulting value into the corresponding parameter.

```c
void set_to_zero(int value) {
    value = 0;
}
```

Calling:

```c
int number = 12;
set_to_zero(number);
```

does not change `number`.

The function changes only its local parameter copy.

To let a function modify the caller's object, I pass a pointer value:

```c
#include <stdbool.h>

bool set_to_zero(int *value) {
    if (value == NULL) {
        return false;
    }

    *value = 0;
    return true;
}
```

The pointer is still passed by value.

What changes is that the copied pointer designates the caller's object.

### Dry Run: Swapping Two Integers

```c
#include <stdbool.h>

bool swap_ints(int *left, int *right) {
    if (left == NULL || right == NULL) {
        return false;
    }

    int temporary = *left;
    *left = *right;
    *right = temporary;
    return true;
}
```

Take:

```txt
a = 4
b = 9
```

Call:

```c
swap_ints(&a, &b);
```

The dry run is:

| Step | `a` | `b` | `temporary` |
|---|---:|---:|---:|
| Start | 4 | 9 | not initialized |
| `temporary = *left` | 4 | 9 | 4 |
| `*left = *right` | 9 | 9 | 4 |
| `*right = temporary` | 9 | 4 | 4 |

The pointers identify which caller-owned objects are modified.

If both pointers designate the same object, the function remains valid:

```c
swap_ints(&a, &a);
```

The value is copied out and then copied back.

Aliasing changes the dry run, so it belongs in the analysis.

## 9. Output Parameters Need Contracts

A function can return one value directly and produce additional results through pointer parameters.

```c
#include <stdbool.h>
#include <stddef.h>

bool find_first(
    const int *array,
    size_t length,
    int target,
    size_t *index_out
) {
    if (index_out == NULL) {
        return false;
    }

    if (length > 0 && array == NULL) {
        return false;
    }

    for (size_t i = 0; i < length; ++i) {
        if (array[i] == target) {
            *index_out = i;
            return true;
        }
    }

    return false;
}
```

The contract should answer:

- Is `array == NULL` permitted when `length == 0`?
- Must `index_out` always be non-null?
- Is `*index_out` changed when the target is absent?
- May `index_out` point inside `array`?

For the implementation above:

- a null array is accepted only for zero length
- `index_out` must be non-null
- the output is written only when the target is found

The caller must not read an output that the function did not produce:

```c
size_t index;

if (find_first(values, count, target, &index)) {
    printf("found at %zu\n", index);
}
```

An output parameter is not merely a pointer.

It is part of an interface agreement about readable and writable storage.

## 10. Arrays Are Objects, but They Often Convert to Pointers

Consider:

```c
int values[4] = {10, 20, 30, 40};
```

`values` is an array object containing four `int` elements.

In most expressions, the array expression is converted to a pointer to its first element.

Therefore:

```c
int *first = values;
```

has the same result as:

```c
int *first = &values[0];
```

This conversion is often called array-to-pointer decay.

It does not mean arrays and pointers are the same type.

The difference appears immediately with `sizeof`:

```c
#include <stdio.h>

int main(void) {
    int values[4] = {10, 20, 30, 40};
    int *pointer = values;

    printf("%zu\n", sizeof values);
    printf("%zu\n", sizeof pointer);

    return 0;
}
```

`sizeof values` is the size of the complete array.

`sizeof pointer` is the size of one pointer.

On a machine with four-byte `int` and eight-byte pointers, the output may be:

```txt
16
8
```

Those sizes are common, not universal.

### Why Length Must Travel Separately

In a function parameter, these declarations are adjusted to pointer parameters:

```c
void inspect(int values[]);
void inspect(int values[100]);
void inspect(int *values);
```

For this purpose, all three declare a function receiving `int *`.

The called function does not recover the array's element count from that pointer.

This is wrong:

```c
size_t bad_length(int values[]) {
    return sizeof values / sizeof values[0];
}
```

Inside the function, `sizeof values` measures the pointer parameter.

The robust interface carries length explicitly:

```c
void inspect(const int *values, size_t length);
```

Pointer plus length is one of the most important interface patterns in C.

## 11. Indexing Is Defined Through Pointer Arithmetic

For a valid array access:

```c
values[i]
```

C defines the subscript operation in terms of pointer arithmetic:

```c
*(values + i)
```

If `values` points to an `int`, then:

```c
values + 1
```

advances by one `int`, not by one byte.

Conceptually:

```txt
values + 0    values + 1    values + 2    values + 3
    |             |             |             |
    v             v             v             v
+--------+    +--------+    +--------+    +--------+
|   10   |    |   20   |    |   30   |    |   40   |
+--------+    +--------+    +--------+    +--------+
```

The scaling by `sizeof(int)` is part of typed pointer arithmetic.

This strange-looking expression is valid:

```c
2[values]
```

because it also becomes:

```c
*(2 + values)
```

It is legal but hostile to readers.

I should write `values[2]`.

## 12. Pointer Arithmetic Has Strict Boundaries

For an array of `n` elements, pointers may be formed to:

- any element from index `0` through `n - 1`
- the one-past position at index `n`

The one-past pointer is useful as an exclusive boundary:

```c
const int *current = values;
const int *end = values + length;

while (current != end) {
    printf("%d\n", *current);
    ++current;
}
```

`end` may be compared with pointers into the same array.

It must not be dereferenced.

```c
printf("%d\n", *end); /* Wrong. */
```

Forming pointers arbitrarily before the first element or beyond one-past is not a valid way to traverse memory.

Pointer subtraction is meaningful when both pointers refer into the same array object, including one-past:

```c
#include <stddef.h>

ptrdiff_t distance = end - current;
```

`ptrdiff_t` is a signed integer type designed for pointer differences.

Relational comparisons such as `<` and `>` are likewise intended for positions within the same array.

Addresses from unrelated allocations do not form one portable ordered coordinate system for application logic.

### Empty Ranges Need Special Care

The half-open range:

```txt
[begin, end)
```

is empty when `begin == end`.

If `length == 0`, a function can often avoid dereferencing `array`.

But I must not perform pointer arithmetic on a null pointer merely because the offset is zero.

A safe loop can use indices:

```c
for (size_t i = 0; i < length; ++i) {
    use(array[i]);
}
```

Under a contract that permits `array == NULL` only when `length == 0`, the body never executes for the null case.

## 13. `const` Describes Allowed Access

This parameter:

```c
const int *values
```

means the function must not modify the `int` objects through `values`.

It may move the pointer:

```c
++values;
```

but it may not write:

```c
*values = 10; /* Constraint violation. */
```

These declarations differ:

```c
const int *a;
int const *b;
int *const c = some_pointer;
const int *const d = some_pointer;
```

They mean:

- `a` points to a read-only-through-`a` `int`
- `b` means the same as `a`
- `c` is a non-reassignable pointer to writable `int`
- `d` is a non-reassignable pointer to read-only-through-`d` `int`

A useful reading rule is to begin at the name:

```txt
int *const c
     ^^^^^
```

`c` is const.

It is a const pointer.

```txt
const int *a
^^^^^
```

The pointed-to `int` is const through this access path.

`const` does not guarantee that nobody else can modify the object.

```c
int value = 5;
const int *read_only_view = &value;

value = 8; /* Valid: value itself was not declared const. */
```

It gives a local access guarantee through the qualified pointer.

Read-only input parameters should normally use `const`.

The type then documents and enforces part of the function contract.

## 14. Aliasing Means Multiple Access Paths

Two pointers alias when they designate the same object or overlapping storage.

```c
int value = 10;
int *first = &value;
int *second = &value;

*first = 20;
printf("%d\n", *second); /* 20 */
```

The write through `first` is visible through `second` because both reach the same object.

Aliasing matters for correctness:

```c
void copy_then_clear(int *destination, int *source) {
    *destination = *source;
    *source = 0;
}
```

With distinct objects:

```txt
before: destination = 3, source = 8
after:  destination = 8, source = 0
```

With both pointers equal:

```txt
before: object = 8
copy:   object = 8
clear:  object = 0
```

The final result is different.

An interface should either:

- support aliasing deliberately
- reject overlapping arguments
- document that the arguments must not overlap

Library functions show why the distinction matters.

`memcpy` requires that source and destination do not overlap.

`memmove` supports overlap by preserving the logical source bytes during the move.

The more mutable aliases exist, the harder it becomes to reason about state.

Ownership and controlled mutation reduce that problem.

## 15. A Pointer Is Not Ownership

A pointer answers:

> Where can I access an object?

It does not answer:

> Who is responsible for ending this object's lifetime?

These are different relationships.

An owning pointer is responsible for eventually releasing allocated storage.

A borrowed pointer temporarily accesses storage owned elsewhere.

Consider:

```c
int *data = malloc(10 * sizeof *data);
```

If allocation succeeds, `data` is initially the owning pointer.

This alias borrows access:

```c
int *third = &data[2];
```

`third` must not be passed to `free`.

It does not point to the start of the allocated block.

After:

```c
free(data);
```

both `data` and `third` are dangling pointer values.

Assigning:

```c
data = NULL;
```

prevents accidental reuse through `data`.

It does not repair `third`.

Every alias must stop being used when the owned object's lifetime ends.

This is why ownership is a whole-program reasoning problem, not merely a call to `free`.

## 16. Dynamic Allocation Starts With a Size Calculation

`malloc` allocates a requested number of bytes:

```c
#include <stdlib.h>

int *value = malloc(sizeof *value);
```

In C, `void *` converts to an object-pointer type without an explicit cast.

I should not write:

```c
int *value = (int *)malloc(sizeof(int));
```

The cast adds noise and can hide a missing declaration of `malloc` in poorly configured code.

Using the pointed-to expression in `sizeof` is resilient:

```c
int *value = malloc(sizeof *value);
```

If the pointer type changes later, the allocation expression changes with it.

For an array:

```c
size_t count = 100;
int *values = malloc(count * sizeof *values);
```

The multiplication is part of the correctness proof.

If `count * sizeof *values` overflows `size_t`, `malloc` receives a smaller number than intended.

Writing `count` elements would then exceed the allocation.

### Checked Multiplication

```c
#include <stdbool.h>
#include <stdint.h>

bool checked_array_bytes(
    size_t count,
    size_t element_size,
    size_t *bytes_out
) {
    if (bytes_out == NULL) {
        return false;
    }

    if (element_size != 0 && count > SIZE_MAX / element_size) {
        return false;
    }

    *bytes_out = count * element_size;
    return true;
}
```

The guard rearranges the unsafe condition.

Instead of multiplying first, it asks whether:

```txt
count > largest representable size / element size
```

If true, multiplication would overflow.

Only after the check is multiplication safe.

## 17. Allocation Failure Is an Ordinary Branch

`malloc` returns a null pointer when it cannot satisfy a nonzero allocation request.

```c
size_t bytes;

if (!checked_array_bytes(count, sizeof *values, &bytes)) {
    return false;
}

int *values = malloc(bytes);

if (values == NULL && bytes != 0) {
    return false;
}
```

An allocation request of zero bytes is a special case.

The implementation may return null or a non-null pointer that still must not be dereferenced as an element.

Interfaces are easier to reason about when they define their own zero-size policy.

For example:

```c
int *allocate_ints(size_t count) {
    if (count == 0) {
        return NULL;
    }

    if (count > SIZE_MAX / sizeof(int)) {
        return NULL;
    }

    return malloc(count * sizeof(int));
}
```

Now null consistently means either:

- zero elements requested
- allocation failed
- size calculation overflowed

That ambiguity may be acceptable if the caller already knows `count`.

If the caller must distinguish the causes, the interface needs a separate status result.

## 18. Initialized and Uninitialized Storage

`malloc` does not initialize the allocated bytes.

```c
int *values = malloc(count * sizeof *values);
```

After successful allocation, reading `values[i]` before writing a valid value is wrong.

A safe initialization loop is:

```c
for (size_t i = 0; i < count; ++i) {
    values[i] = 0;
}
```

`calloc` allocates storage and initializes all bits to zero:

```c
int *values = calloc(count, sizeof *values);
```

It also accepts the element count and element size separately, allowing the implementation to detect an impossible total size.

All-bits-zero represents integer zero for the integer types used in ordinary C implementations.

I should still understand the precise operation:

> `calloc` zeroes the object representation bytes.

It does not invoke constructors, because C has no constructors.

It does not establish a domain-specific invariant merely because the fields are zero.

For a structure where `0` is not a valid state code or where a non-null pointer is required, zero-filled storage is not fully initialized application state.

## 19. `free` Ends the Allocated Lifetime

```c
free(values);
```

After `free`:

- the allocated object's lifetime has ended
- its stored values may no longer be read
- the storage may no longer be written
- pointers into the block are dangling
- the same allocation must not be freed again

This is a use-after-free:

```c
free(values);
printf("%d\n", values[0]); /* Wrong. */
```

This is a double free:

```c
free(values);
free(values); /* Wrong. */
```

This pattern helps for one owning variable:

```c
free(values);
values = NULL;
```

Calling:

```c
free(NULL);
```

is valid and does nothing.

Therefore cleanup functions can often be written without a branch:

```c
void release_ints(int **values) {
    if (values == NULL) {
        return;
    }

    free(*values);
    *values = NULL;
}
```

However, accepting `int **` just to null the caller's pointer is not universally necessary.

For a structure owner, a `destroy` function can reset all state fields and make repeated destruction harmless.

## 20. Losing the Owner Causes a Leak

This leaks memory:

```c
int *values = malloc(100 * sizeof *values);
values = malloc(200 * sizeof *values);
```

If the first allocation succeeded, its only owning pointer was overwritten.

The program can no longer pass that allocation's starting address to `free`.

The second allocation may also fail, leaving `values == NULL` and losing both useful state and the first allocation.

The general rule is:

> Do not overwrite the only owning pointer until the replacement operation has succeeded and the old ownership has been resolved.

This rule becomes crucial with `realloc`.

## 21. `realloc` Can Move the Allocation

`realloc` changes the size of an allocated block:

```c
int *resized = realloc(values, new_count * sizeof *values);
```

If it succeeds:

- it returns a pointer to the resized block
- the block may be at a different address
- old element values are preserved up to the smaller of the old and new byte sizes
- the old pointer value must no longer be used

If it fails for a nonzero requested size:

- it returns null
- the original allocation remains valid and owned by the original pointer

This is dangerous:

```c
values = realloc(values, new_bytes);
```

On failure, `values` becomes null and the original allocation is leaked.

Use a temporary pointer:

```c
int *resized = realloc(values, new_bytes);

if (resized == NULL) {
    /* values still owns the original allocation. */
    return false;
}

values = resized;
```

Any pointer into the old allocation must be considered invalid after successful `realloc`, even if the returned address happens to compare equal.

For example:

```c
int *element = &values[3];
int *resized = realloc(values, new_bytes);
```

After success, recompute:

```c
values = resized;
element = &values[3];
```

Do not continue using the pre-`realloc` interior pointer.

## 22. Structures Group State Into One Object

A structure definition creates a new object type:

```c
typedef struct {
    int x;
    int y;
} Point;
```

Now:

```c
Point point = {3, 7};
```

creates one `Point` object with two members.

Access members with `.`:

```c
point.x = 10;
printf("(%d, %d)\n", point.x, point.y);
```

A designated initializer states the mapping explicitly:

```c
Point point = {
    .x = 3,
    .y = 7,
};
```

Designators are valuable when:

- the structure has several same-typed fields
- field order may change
- omitted members should be zero-initialized
- the names carry more meaning than positions

A `struct` value can be copied by assignment:

```c
Point first = {.x = 3, .y = 7};
Point second = first;

second.x = 99;
```

`first.x` remains `3`.

The member values were copied into a distinct structure object.

Arrays cannot be assigned directly in C, but structures—including structures containing arrays—can.

## 23. Structure Layout Includes Alignment and Padding

Members appear in memory in declaration order.

The implementation may insert unnamed padding bytes between members or after the final member to satisfy alignment requirements.

```c
typedef struct {
    char tag;
    int value;
    char active;
} Record;
```

A possible layout is:

```txt
offset 0        offsets 1..3       offsets 4..7      offset 8     9..11
+--------+-----------------------+----------------+------------+---------+
| tag    | padding               | value          | active     | padding |
+--------+-----------------------+----------------+------------+---------+
```

On such an implementation:

```txt
sizeof(Record) == 12
```

But the exact size is not guaranteed by the source alone.

I can inspect member offsets:

```c
#include <stddef.h>
#include <stdio.h>

typedef struct {
    char tag;
    int value;
    char active;
} Record;

int main(void) {
    printf("sizeof Record: %zu\n", sizeof(Record));
    printf("tag offset: %zu\n", offsetof(Record, tag));
    printf("value offset: %zu\n", offsetof(Record, value));
    printf("active offset: %zu\n", offsetof(Record, active));
    return 0;
}
```

Member order can change the amount of padding:

```c
typedef struct {
    int value;
    char tag;
    char active;
} CompactRecord;
```

This may be smaller, but I should measure on the target implementation.

I should not reorder fields blindly when:

- a binary protocol fixes offsets
- an external ABI fixes layout
- serialized data expects a specific format
- cache access patterns favor another grouping
- source clarity would suffer

Memory layout is an engineering tradeoff, not a “sort fields by size” ritual.

### Padding Is Not Data

Structure assignment copies the member values correctly.

Comparing arbitrary structures with `memcmp` is usually wrong:

```c
memcmp(&left, &right, sizeof left)
```

Padding bytes may have indeterminate or different representations even when every member value compares equal.

Compare members according to the structure's logical equality rule.

## 24. Structure Pointers Use `->`

Given:

```c
Point point = {.x = 3, .y = 7};
Point *pointer = &point;
```

This:

```c
pointer->x
```

means:

```c
(*pointer).x
```

The parentheses are necessary in the expanded form because `.` binds more tightly than unary `*`.

This is wrong:

```c
*pointer.x
```

It attempts to use `.` on `pointer` before dereferencing.

A mutation function can make its requirements explicit:

```c
#include <stdbool.h>

typedef struct {
    int x;
    int y;
} Point;

bool translate(Point *point, int dx, int dy) {
    if (point == NULL) {
        return false;
    }

    point->x += dx;
    point->y += dy;
    return true;
}
```

A read-only function should use a pointer to const:

```c
long long squared_distance(const Point *point) {
    if (point == NULL) {
        return 0;
    }

    long long x = point->x;
    long long y = point->y;
    return x * x + y * y;
}
```

The wider intermediate type reduces overflow risk relative to multiplying two `int` values.

The function's null behavior still needs documentation; returning zero for null may be ambiguous with the origin.

A status-plus-output interface may be better when ambiguity matters.

## 25. Structure Invariants Matter More Than Individual Fields

Consider a dynamic integer buffer:

```c
typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntBuffer;
```

The fields are not independent.

A useful invariant is:

```txt
0 <= size <= capacity
```

Additionally:

- if `capacity == 0`, `data == NULL`
- if `capacity > 0`, `data` points to an allocation for at least `capacity` `int`
- elements in `[0, size)` hold initialized values
- positions in `[size, capacity)` are allocated but not logically part of the buffer
- the buffer exclusively owns `data`

This diagram distinguishes logical size from allocation capacity:

```txt
data
 |
 v
+----+----+----+----+----+----+----+----+
| 11 | 24 | 35 | 42 | ?? | ?? | ?? | ?? |
+----+----+----+----+----+----+----+----+
|<------ size ------>|
|<------------- capacity --------------->|
```

The question marks mean:

> Storage exists, but these slots are not initialized logical elements.

Reading them as buffer elements is wrong.

Every operation must preserve the whole invariant.

For example, a successful push must:

1. ensure `size < capacity`
2. write the new value to `data[size]`
3. increment `size`

Changing `size` before the write would temporarily claim an uninitialized element.

If a later operation failed, the structure could be left invalid.

## 26. A Complete Owning Structure

The following implementation is intentionally small but handles the central failure paths.

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntBuffer;

void int_buffer_init(IntBuffer *buffer) {
    if (buffer == NULL) {
        return;
    }

    buffer->data = NULL;
    buffer->size = 0;
    buffer->capacity = 0;
}

void int_buffer_destroy(IntBuffer *buffer) {
    if (buffer == NULL) {
        return;
    }

    free(buffer->data);

    /* Restore the same valid empty state created by init. */
    buffer->data = NULL;
    buffer->size = 0;
    buffer->capacity = 0;
}

static bool int_buffer_bytes(size_t capacity, size_t *bytes_out) {
    if (bytes_out == NULL) {
        return false;
    }

    if (capacity > SIZE_MAX / sizeof(int)) {
        return false;
    }

    *bytes_out = capacity * sizeof(int);
    return true;
}

bool int_buffer_reserve(IntBuffer *buffer, size_t new_capacity) {
    if (buffer == NULL) {
        return false;
    }

    if (new_capacity <= buffer->capacity) {
        return true;
    }

    size_t new_bytes;

    if (!int_buffer_bytes(new_capacity, &new_bytes)) {
        return false;
    }

    int *resized = realloc(buffer->data, new_bytes);

    if (resized == NULL) {
        /*
         * realloc failure leaves buffer->data and every structure
         * field unchanged, so the old invariant is preserved.
         */
        return false;
    }

    buffer->data = resized;
    buffer->capacity = new_capacity;
    return true;
}

bool int_buffer_push(IntBuffer *buffer, int value) {
    if (buffer == NULL) {
        return false;
    }

    if (buffer->size == buffer->capacity) {
        size_t next_capacity;

        if (buffer->capacity == 0) {
            next_capacity = 1;
        } else {
            if (buffer->capacity > SIZE_MAX / 2) {
                return false;
            }

            next_capacity = buffer->capacity * 2;
        }

        if (!int_buffer_reserve(buffer, next_capacity)) {
            return false;
        }
    }

    buffer->data[buffer->size] = value;
    ++buffer->size;
    return true;
}

bool int_buffer_get(
    const IntBuffer *buffer,
    size_t index,
    int *value_out
) {
    if (buffer == NULL || value_out == NULL) {
        return false;
    }

    if (index >= buffer->size) {
        return false;
    }

    *value_out = buffer->data[index];
    return true;
}

bool int_buffer_clone(IntBuffer *destination, const IntBuffer *source) {
    if (destination == NULL || source == NULL) {
        return false;
    }

    if (destination == source) {
        return true;
    }

    /*
     * Build a complete temporary owner first. The destination is
     * changed only after every fallible operation succeeds.
     */
    IntBuffer copy = {
        .data = NULL,
        .size = 0,
        .capacity = 0,
    };

    if (source->size > 0) {
        if (!int_buffer_reserve(&copy, source->size)) {
            return false;
        }

        memcpy(
            copy.data,
            source->data,
            source->size * sizeof *source->data
        );

        copy.size = source->size;
    }

    int_buffer_destroy(destination);
    *destination = copy;
    return true;
}
```

The public contract includes preconditions not expressible by the type system:

- each `IntBuffer` passed in must already be initialized
- its fields must satisfy the invariant
- the caller must not copy an owning buffer with ordinary assignment
- no concurrent unsynchronized mutation may occur

### Why Initialization Is a Real Operation

This is valid:

```c
IntBuffer buffer;
int_buffer_init(&buffer);
```

This is also valid:

```c
IntBuffer buffer = {0};
```

For this particular structure, all-zero state is the chosen empty invariant.

This is not valid:

```c
IntBuffer buffer;
int_buffer_push(&buffer, 10);
```

The uninitialized fields contain indeterminate values.

The function cannot distinguish those bytes from a legitimate state.

### Why Destruction Resets the Fields

After:

```c
int_buffer_destroy(&buffer);
```

the object `buffer` itself still exists if it was automatic or static.

Only its owned allocation has ended.

Resetting the fields restores a valid empty buffer, so:

- accidental reads are easier to detect
- repeated destruction is harmless
- the buffer can be reused

### Why Clone Uses a Temporary Owner

Suppose allocation fails while cloning.

The destination should retain its old valid value.

The function first constructs `copy`.

Only after construction succeeds does it:

1. destroy the destination's old allocation
2. transfer the complete temporary state into the destination

This is a commit-or-rollback pattern:

> Either the full operation succeeds, or the original destination remains unchanged.

Failure-path invariants are part of data-structure correctness.

## 27. Dry Run: Growing the Buffer

Start with:

```txt
data = NULL
size = 0
capacity = 0
```

Push `10`:

1. `size == capacity`, so growth is required.
2. Zero capacity grows to `1`.
3. `realloc(NULL, sizeof(int))` behaves like allocation.
4. The value is written at index `0`.
5. `size` becomes `1`.

```txt
size = 1, capacity = 1
+----+
| 10 |
+----+
```

Push `20`:

1. `size == capacity`, so capacity doubles to `2`.
2. The old value is preserved by successful `realloc`.
3. `20` is written at index `1`.
4. `size` becomes `2`.

```txt
size = 2, capacity = 2
+----+----+
| 10 | 20 |
+----+----+
```

Push `30`:

1. Capacity doubles from `2` to `4`.
2. Existing values remain at indices `0` and `1`.
3. `30` is written at index `2`.
4. `size` becomes `3`.

```txt
size = 3, capacity = 4
+----+----+----+----+
| 10 | 20 | 30 | ?? |
+----+----+----+----+
```

If growth fails during any push:

- `realloc` leaves the old allocation valid
- `reserve` changes no fields
- `push` returns `false`
- `size`, `capacity`, and existing elements remain unchanged

The failure dry run matters as much as the successful one.

## 28. Shallow Copy and Deep Copy

Ordinary structure assignment copies each member value.

For an owning structure:

```c
IntBuffer first = {0};
int_buffer_push(&first, 10);

IntBuffer second = first;
```

the result is:

```txt
first.data  ----+
                |
                v
             +----+
             | 10 |
             +----+
                ^
                |
second.data ----+
```

The pointer value was copied.

The allocated elements were not duplicated.

Both structures now appear to own the same block.

If both are destroyed, the program attempts to free the allocation twice.

If one is destroyed, the other contains a dangling pointer.

This is a shallow copy.

A deep copy creates a distinct allocation and copies the logical elements:

```txt
first.data  ------> +----+
                    | 10 |
                    +----+

second.data ------> +----+
                    | 10 |
                    +----+
```

The `int_buffer_clone` function performs a deep copy.

The correct copy policy depends on the type:

- plain value structure: ordinary assignment may be enough
- shared immutable storage: shallow copy may be deliberate
- reference-counted storage: shallow copy must update the count
- unique owner: ordinary copying must be forbidden by convention
- independent owner: provide an explicit deep-copy operation

C does not automatically run copy constructors or destructors.

The programmer must make ownership semantics visible through interfaces and discipline.

## 29. Self-Referential Structures Store Pointers

A linked-list node needs to refer to another node of the same type.

This cannot contain another complete node directly:

```c
struct Node {
    int value;
    struct Node next; /* Impossible: infinite size. */
};
```

If each node physically contained another complete node, calculating the structure size would never terminate.

Use a pointer:

```c
typedef struct Node {
    int value;
    struct Node *next;
} Node;
```

A pointer has a fixed size independent of the pointed-to structure's complete size.

Conceptually:

```txt
+---------+---------+     +---------+---------+     +---------+------+
| value 5 | next -------->| value 8 | next -------->| value 13| NULL |
+---------+---------+     +---------+---------+     +---------+------+
```

The structure definition establishes layout.

It does not allocate any nodes.

Each dynamically created node still needs:

```c
Node *node = malloc(sizeof *node);
```

and each successfully allocated node eventually needs exactly one `free`.

The linked-list chapter will turn this memory shape into complete operations and invariants.

## 30. Double Pointers Represent Access to a Pointer Object

An `int **` is a pointer to an `int *` object.

It is not inherently “a two-dimensional array.”

Consider:

```c
#include <stdbool.h>
#include <stdlib.h>

bool allocate_int(int **result, int initial_value) {
    if (result == NULL) {
        return false;
    }

    int *created = malloc(sizeof *created);

    if (created == NULL) {
        return false;
    }

    *created = initial_value;
    *result = created;
    return true;
}
```

The caller uses:

```c
int *value = NULL;

if (!allocate_int(&value, 42)) {
    /* Handle failure. */
}
```

The levels are:

```txt
&value       value          *value
   |           |               |
   v           v               v
+--------+   +---------+     +----+
| int ** |-->| int *   |---->| 42 |
+--------+   +---------+     +----+
```

Inside `allocate_int`:

- `result` is a copied pointer to the caller's pointer object
- `*result` is the caller's `value`
- `**result` would be the allocated integer after assignment

The temporary `created` pointer ensures the caller's output is not changed on allocation failure.

Again, the operation either succeeds completely or preserves the old state.

## 31. Two-Dimensional Arrays Are Not `int **`

This object:

```c
int matrix[3][4];
```

is an array of three elements.

Each element is an array of four `int`.

Its layout is contiguous:

```txt
row 0: [int][int][int][int]
row 1: [int][int][int][int]
row 2: [int][int][int][int]
```

When `matrix` converts in an expression, it becomes a pointer to its first row:

```c
int (*row_pointer)[4] = matrix;
```

It does not become `int **`.

A function receiving this fixed-column layout can be:

```c
#include <stddef.h>

void clear_matrix(
    size_t rows,
    size_t columns,
    int matrix[rows][columns]
) {
    for (size_t row = 0; row < rows; ++row) {
        for (size_t column = 0; column < columns; ++column) {
            matrix[row][column] = 0;
        }
    }
}
```

After parameter adjustment, `matrix` is a pointer to an array of `columns` `int`.

This particular variable-length-array interface requires positive dimensions.

An interface that must represent a zero-row or zero-column matrix should use a flat pointer plus explicit dimensions and define its empty-state contract separately.

An `int **` representation is different:

```txt
row pointers                 separately located rows
+---------+                 +----+----+----+----+
| row 0 ------------------->|    |    |    |    |
+---------+                 +----+----+----+----+
| row 1 -----------+        +----+----+----+----+
+---------+        +------->|    |    |    |    |
| row 2 -----+              +----+----+----+----+
+---------+   |             +----+----+----+----+
              +------------>|    |    |    |    |
                            +----+----+----+----+
```

The rows may be separate allocations and need not be adjacent.

The two representations require different allocation, indexing, and cleanup logic.

## 32. Alignment Is a Real Constraint

Different types may require addresses divisible by different powers of two.

An implementation might require:

- `char`: one-byte alignment
- `int`: four-byte alignment
- `double`: eight-byte alignment

These are examples, not universal requirements.

`malloc` returns storage suitably aligned for ordinary object types.

But arbitrary byte offsets may destroy that alignment:

```c
unsigned char *bytes = malloc(32);
int *value = (int *)(bytes + 1);
```

`bytes + 1` may not satisfy `int` alignment.

Dereferencing `value` may have undefined behavior.

This is one reason serialization code should not cast arbitrary byte positions to structure pointers.

A safer approach copies bytes into a properly aligned object and decodes a defined format explicitly.

Alignment also explains some structure padding.

The compiler places members where their alignment requirements can be met and rounds the structure size so array elements of that structure remain aligned.

## 33. Effective Type and Aliasing Rules

C permits object bytes to be inspected through `char`, `signed char`, or `unsigned char` access.

It does not generally permit storage containing one object type to be read through an unrelated object-pointer type.

This is not a portable way to inspect floating-point bits:

```c
float value = 1.0f;
unsigned int bits = *(unsigned int *)&value; /* Problematic aliasing. */
```

A defined byte-copy approach is:

```c
#include <stdint.h>
#include <string.h>

uint32_t float_bits(float value) {
    uint32_t bits = 0;

    _Static_assert(
        sizeof bits == sizeof value,
        "float_bits requires 32-bit float"
    );

    memcpy(&bits, &value, sizeof bits);
    return bits;
}
```

The static assertion makes the representation-size precondition explicit.

This does not claim that every machine uses the same floating-point encoding.

It only copies the local representation into an integer object of equal size.

Low-level code needs both:

- correct byte movement
- a defined interpretation of the bytes

## 34. Undefined Behavior Is Not a Predictable Error Value

Examples of undefined behavior in this domain include:

- dereferencing null
- accessing an object after its lifetime ends
- indexing outside an array
- reading an uninitialized automatic object
- signed integer overflow
- double-freeing an allocation
- freeing a pointer that was not returned as an allocation start
- violating required alignment
- modifying a string literal

Undefined behavior does not mean:

> The program always crashes at this line.

It means the C language no longer places the ordinary required constraints on the program's behavior.

The program may:

- appear to work
- produce a wrong value
- crash later
- change behavior under optimization
- expose a security vulnerability

“It worked once” is not evidence that undefined behavior is safe.

Correct pointer reasoning aims to prove that invalid operations are unreachable.

## 35. Common Memory Bugs

### Uninitialized Pointer

```c
int *pointer;
*pointer = 10;
```

`pointer` has not been given a valid target.

### Null Dereference

```c
int *pointer = NULL;
printf("%d\n", *pointer);
```

Null represents no object.

### Out-of-Bounds Access

```c
int values[4] = {0};
values[4] = 10;
```

Valid indices are `0` through `3`.

The one-past position cannot be dereferenced.

### Use After Free

```c
int *value = malloc(sizeof *value);
free(value);
*value = 10;
```

The allocated lifetime has ended.

### Double Free

```c
free(value);
free(value);
```

The second call does not receive a currently allocated block.

### Invalid Free

```c
int values[4];
free(values);
```

Automatic array storage was not obtained from an allocation function.

This is also invalid:

```c
int *values = malloc(4 * sizeof *values);
free(values + 1);
```

Only the allocation's returned start pointer may be released.

### Memory Leak

```c
void leak(void) {
    int *value = malloc(sizeof *value);
    (void)value;
}
```

The function loses its owning pointer without releasing the allocation.

### Size Overflow

```c
int *values = malloc(count * sizeof *values);
```

The multiplication must be checked before allocation when `count` is untrusted or may be large.

### Shallow Copy of an Owner

```c
IntBuffer second = first;
```

Both structures now contain the same owning pointer without a shared-ownership protocol.

### Stale Interior Pointer

```c
int *element = &buffer.data[2];
int_buffer_push(&buffer, 99);
```

If push reallocates, `element` becomes invalid.

Long-lived pointers into resizable storage are dangerous.

An index can survive relocation:

```c
size_t index = 2;
int_buffer_push(&buffer, 99);
int *element = &buffer.data[index];
```

The index must still be checked against the current logical size.

## 36. Common Misconceptions

### “A Pointer Is Just an Integer Address”

No.

Pointer validity depends on object lifetime, type, alignment, bounds, and the permitted operation.

Treating it as only a number discards the rules that make dereference meaningful.

### “Arrays and Pointers Are the Same”

No.

An array is an object containing contiguous elements.

A pointer is an object that can designate another object.

Array expressions often convert to pointers, which creates the confusion.

### “`sizeof(pointer)` Gives the Allocation Size”

No.

It gives the size of the pointer object.

C pointers do not carry a portable allocation length.

The program must store length or capacity separately.

### “Passing a Pointer Means Pass-by-Reference”

C still passes the pointer value by value.

The copied pointer can be used to modify a shared pointed-to object.

### “Non-Null Means Safe”

No.

A dangling, misaligned, out-of-bounds, or incorrectly typed pointer can be non-null.

### “Setting One Pointer to Null Fixes Every Alias”

No.

Other pointers to the released object remain dangling.

### “`malloc` Returns Initialized Objects”

No.

Successful `malloc` returns uninitialized storage.

The program must establish valid values before reading them.

### “`free` Erases the Pointer”

No.

`free` ends the allocation's lifetime.

The pointer variable still contains a now-invalid pointer value until reassigned.

### “Memory Leaks Are the Only Allocation Bug”

No.

Use-after-free, double free, invalid free, overflowed sizes, stale aliases, and partial initialization can be more immediately destructive.

### “Structure Assignment Always Makes an Independent Copy”

It makes a new structure object and copies member values.

If a member is a pointer, the address is copied, not the pointed-to allocation.

### “`int **` Is a Two-Dimensional Array”

Not inherently.

It points to an `int *` object.

A true two-dimensional array decays to a pointer to a row array, not to `int **`.

### “The Stack Is Always Small and the Heap Is Always Large”

Those are common implementation tendencies, not a complete language rule or design method.

The correct choice depends on lifetime, size, ownership, allocation cost, and environment limits.

### “If Sanitizers Find Nothing, the Program Is Proven Correct”

No.

Dynamic tools observe executed paths.

They strengthen testing but do not replace invariants, reviews, or complete reasoning.

## 37. Edge Cases to Check Deliberately

### Zero Elements

Define whether:

```txt
data = NULL, size = 0
```

is accepted.

Do not dereference or perform invalid pointer arithmetic merely because a loop has zero iterations.

### One Element

One-element allocations expose:

- off-by-one loop bounds
- incorrect one-past dereference
- bad growth transitions from zero capacity

### Maximum Sizes

Check:

- `count * sizeof(element)` overflow
- capacity doubling overflow
- `size + 1` overflow
- whether a pointer difference fits `ptrdiff_t`

### Allocation Failure

Every allocating operation should specify:

- its return status
- whether old state is preserved
- who cleans up partial work
- which outputs remain unchanged

### Partial Construction

If a structure owns several allocations, cleanup must handle any prefix of successful initialization.

A useful pattern is:

1. initialize every owner field to null
2. acquire resources one at a time
3. on failure, call one cleanup function that tolerates null fields

### Aliased Arguments

Ask whether two pointer parameters may designate:

- the same object
- overlapping arrays
- a field inside the other argument

If not, document and enforce what can be checked.

### Self-Assignment

An ownership operation such as cloning should handle:

```c
int_buffer_clone(&buffer, &buffer);
```

Destroying the destination before recognizing self-assignment would destroy the source too.

### Empty Versus Uninitialized

These are not the same:

```c
IntBuffer empty = {0};
IntBuffer unknown;
```

The first has a defined empty state.

The second has no established invariant.

### Reallocation and Borrowed Pointers

Any operation that may grow a container may invalidate:

- element pointers
- iterators represented as pointers
- pointers to structure-internal arrays

The interface should say when borrowed views expire.

### Structure Padding

Do not assume:

- no padding exists
- padding bytes are zero
- `sizeof(struct)` equals the sum of member sizes
- `memcmp` gives logical equality
- dumping raw structure bytes creates a portable file format

### Cleanup After Failure

Exercise failure at every allocation point.

Successful-path testing cannot reveal whether partial owners leak or whether old state is corrupted.

## 38. Tools That Expose Memory Errors

Compiler warnings should be enabled aggressively.

With GCC or Clang:

```sh
cc -std=c17 -Wall -Wextra -Wpedantic -Wconversion source.c
```

Warnings are not cosmetic.

They can expose:

- suspicious conversions
- missing declarations
- unused results
- incorrect format strings
- unreachable or inconsistent code

AddressSanitizer can detect many out-of-bounds and lifetime errors:

```sh
cc -std=c17 -Wall -Wextra -Wpedantic \
    -fsanitize=address,undefined -fno-omit-frame-pointer \
    -g source.c -o program
```

Then run:

```sh
./program
```

UndefinedBehaviorSanitizer catches several other invalid operations.

Valgrind's Memcheck can detect leaks and invalid accesses on supported platforms:

```sh
valgrind --leak-check=full --show-leak-kinds=all ./program
```

No single tool catches every error.

A useful workflow is:

1. compile with strong warnings
2. run normal tests
3. run boundary and failure-path tests
4. run with sanitizers
5. run a leak checker where available
6. inspect ownership and invariants manually

Tools provide evidence.

The design still needs a proof story.

## 39. A Repeatable Memory-Reasoning Method

For every pointer or owning structure, I will ask:

1. What object exists?
2. What is its type and logical state?
3. When does its lifetime begin?
4. When does its lifetime end?
5. Which pointer owns it?
6. Which pointers merely borrow it?
7. What are the valid bounds?
8. Which aliases may exist?
9. Can an operation relocate the object?
10. What invariant relates all structure fields?
11. What happens if allocation fails?
12. What cleanup is required on every exit path?
13. Can size arithmetic overflow before allocation?
14. What does empty state mean?
15. Which dynamic tool can exercise the suspected failure?

If I cannot answer those questions, I am not yet ready to manipulate the memory safely.

## 40. Practice Questions

For every code question:

1. draw the relevant objects
2. identify owners and borrowers
3. state lifetimes
4. mark valid bounds
5. identify aliases
6. state the invariant before and after the operation
7. analyze success and failure paths

### Question 1: Values and Addresses

Dry-run:

```c
int a = 5;
int b = 8;
int *p = &a;
int *q = p;

*q = 11;
p = &b;
*p = *q + 1;
```

After every line, record:

- `a`
- `b`
- what `p` points to
- what `q` points to

Explain why reassigning `p` does not reassign `q`.

### Question 2: Return Lifetime

Analyze:

```c
int *make_value(void) {
    int value = 42;
    return &value;
}
```

Explain exactly when the pointed-to object's lifetime ends.

Write two corrected versions:

- one using caller-provided storage
- one using allocated storage with explicit ownership

Compare their failure modes.

### Question 3: Array Decay

Given:

```c
int values[12];
int *pointer = values;
```

Explain each result:

```c
sizeof values
sizeof pointer
sizeof values[0]
&values
values
&values[0]
```

State the type of each address expression.

Explain why `&values + 1` and `values + 1` advance by different byte distances.

### Question 4: One-Past Boundaries

Write a function that sums `[begin, end)` using pointers.

State all preconditions required for:

```c
end - begin
```

Dry-run empty, one-element, and five-element ranges.

Explain why dereferencing `end` is invalid.

### Question 5: Output Preservation

Rewrite this function so `*result` is unchanged on failure:

```c
bool divide(int numerator, int denominator, int *result);
```

Handle:

- null output
- zero denominator
- the signed-overflow case involving the minimum `int`

Explain the order in which checks and writes must occur.

### Question 6: Allocation Overflow

Write:

```c
void *allocate_array(size_t count, size_t element_size);
```

Choose and document a policy for zero-sized requests.

Reject multiplication overflow before calling `malloc`.

Explain why checking the product after multiplication is too late.

### Question 7: Failure-Safe Replacement

Find every bug:

```c
bool grow(int **values, size_t old_count, size_t new_count) {
    *values = realloc(*values, new_count * sizeof **values);

    for (size_t i = old_count; i < new_count; ++i) {
        (*values)[i] = 0;
    }

    return true;
}
```

Rewrite it with:

- null checks
- overflow checks
- a temporary pointer
- preserved old ownership on failure
- a defined policy when `new_count == 0`

### Question 8: Ownership Table

For the following sequence, build a table showing each pointer's state as:

- owner
- borrower
- null
- dangling

```c
int *data = malloc(4 * sizeof *data);
int *middle = data == NULL ? NULL : &data[2];
int *alias = data;
free(data);
data = NULL;
```

Explain why `alias` and `middle` remain dangerous after `data` becomes null.

### Question 9: Structure Padding

For:

```c
typedef struct {
    char first;
    double amount;
    char second;
    int count;
} Example;
```

Use `sizeof`, `_Alignof`, and `offsetof` to inspect the layout on your machine.

Reorder the fields and measure again.

Explain which observations are implementation-specific and which C guarantees remain.

### Question 10: Logical Equality

Write:

```c
bool records_equal(const Record *left, const Record *right);
```

Define how null arguments should behave.

Compare members rather than using `memcmp`.

Then explain a case where two floating-point members make even member-wise equality a domain decision.

### Question 11: Shallow Copy Failure

Dry-run:

```c
IntBuffer first = {0};
int_buffer_push(&first, 10);
IntBuffer second = first;
int_buffer_destroy(&first);
int_buffer_destroy(&second);
```

Mark the exact point where `second.data` becomes dangling.

Explain why the final destruction is invalid.

Replace the assignment with the chapter's clone operation and redraw the memory.

### Question 12: Clone Failure Injection

Imagine `int_buffer_reserve` fails while cloning.

Prove that:

- the source is unchanged
- the destination is unchanged
- no temporary allocation leaks

Then modify `IntBuffer` to own two separately allocated arrays and design a clone algorithm that remains failure-safe after either allocation fails.

### Question 13: Stale Element Pointer

Analyze:

```c
IntBuffer buffer = {0};
int_buffer_push(&buffer, 10);

int *first = &buffer.data[0];
int_buffer_push(&buffer, 20);

printf("%d\n", *first);
```

Why can the final access be invalid?

Rewrite the logic using an index.

State when the index itself could become logically invalid even though it remains an integer.

### Question 14: Self-Referential Nodes

Implement:

```c
Node *node_create(int value, Node *next);
```

Specify whether the new node owns `next` or merely links to it.

Then write a function that releases an entire acyclic chain.

Dry-run:

- an empty chain
- one node
- three nodes

Explain what changes if the chain contains a cycle.

### Question 15: Double-Pointer Removal

Write a function that removes the first node from a singly linked chain:

```c
bool pop_front(Node **head, int *value_out);
```

Require that outputs remain unchanged on failure.

Explain:

- why `Node **head` is needed
- which pointer owns the removed node
- when the head pointer changes
- when the removed node's lifetime ends

### Question 16: Matrix Representations

Implement a `3 x 4` matrix in three ways:

1. a true `int[3][4]`
2. one flat allocation indexed with `row * columns + column`
3. an array of separately allocated row pointers

For each representation, analyze:

- number of allocations
- contiguity
- indexing
- cleanup
- partial-allocation failure
- cache locality

Explain why none of the three can be passed blindly to a function expecting either of the other pointer types.

### Question 17: Partial Construction

Design:

```c
typedef struct {
    char *name;
    int *scores;
    size_t score_count;
} Student;
```

Write:

- an initializer
- a failure-safe constructor-like function
- a destructor
- a deep clone

Handle failure after `name` succeeds but `scores` fails.

State the invariant for every partially and fully initialized state your functions permit.

### Question 18: Serialization Trap

Explain why this is not a portable file format:

```c
fwrite(&record, sizeof record, 1, file);
```

Discuss:

- padding
- byte order
- integer widths
- pointer members
- compiler and ABI differences
- versioning

Design a field-by-field format with explicit widths and byte order.

### Question 19: Const Qualification

For each declaration, state what may change:

```c
int *a;
const int *b;
int *const c = a;
const int *const d = b;
```

Write a function that receives a read-only buffer view and another that mutates elements without reassigning its local base pointer.

Explain what `const` does and does not guarantee about aliases.

### Question 20: Find the Undefined Behavior

Identify every potentially undefined operation:

```c
int *values = malloc(3 * sizeof *values);
int *end = values + 3;

values[0] = 1;
values[1] = 2;
printf("%d\n", values[2]);
printf("%d\n", *end);

free(values);
printf("%d\n", values[0]);
free(values);
```

Include the allocation-failure path in the analysis.

Rewrite the program so every operation is defined.

## 41. Final Check

Before moving on, I should be able to explain:

- why an object is more than a box holding a value
- why lifetime matters even when an address looks unchanged
- why non-null is not enough to prove a pointer valid
- why C still passes pointer arguments by value
- why arrays and pointers are different types
- why pointer arithmetic must remain within one array object
- why one-past pointers may be compared but not dereferenced
- why size and capacity must travel separately from a pointer
- why ownership is not encoded by an ordinary pointer
- why allocation-size arithmetic must be checked before multiplication
- why `realloc` needs a temporary pointer
- why a structure invariant relates all members
- why structure padding defeats raw byte comparison and naive serialization
- why structure assignment is dangerous for owning pointer members
- why a self-referential structure contains a pointer rather than itself
- why `int **` is not automatically a two-dimensional array
- why failure paths must preserve the old valid state
- why sanitizers support reasoning rather than replace it

If I cannot draw the objects, owners, aliases, bounds, and lifetimes for an operation, I do not yet understand the memory behind the data structure.
